# Zulip Integration Guide for Flutter Developers

This guide provides a comprehensive walkthrough for integrating a custom Flutter application with a self-hosted Zulip server. It covers authentication, real-time event handling (heartbeats), sending/receiving messages, and setting up push notifications for a custom app.

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Server Configuration](#server-configuration)
4. [Flutter Implementation](#flutter-implementation)
    - [Dependencies](#dependencies)
    - [ZulipClient Class](#zulipclient-class)
5. [Real-time Events & Heartbeats](#real-time-events--heartbeats)
6. [Push Notifications](#push-notifications)
7. [UI Integration](#ui-integration)

---

## Overview

Zulip is a real-time chat application with a unique threading model. Unlike other chat apps, Zulip organizes conversations into **Streams** (like channels) and **Topics** (sub-threads within streams).

Integration relies on a few key concepts:
- **API Key**: Used for authentication.
- **Event Queue**: A long-polling mechanism to receive real-time updates (messages, reactions, typing status, heartbeats).
- **Push Notifications**: Handled via FCM (Android) or APNs (iOS).

## Prerequisites

- A **self-hosted Zulip server** (version 4.0+ recommended).
- A **Flutter project**.
- Access to the server's `/etc/zulip/settings.py` for push notification configuration.

## Server Configuration

**Crucial for Custom Apps:**
By default, a self-hosted Zulip server is configured to send push notifications to the official Zulip mobile app. Since you are building a **custom app** with your own package ID, you must configure the server to use your own Firebase/APNs credentials.

1.  **Enable Push Notifications**:
    Open `/etc/zulip/settings.py` on your server and set:
    ```python
    ZULIP_SERVICE_PUSH_NOTIFICATIONS = True
    ```

2.  **Configure FCM (Android)**:
    Add your Firebase Server Key (or Service Account JSON path, depending on Zulip version):
    ```python
    # For older Zulip versions using legacy FCM API:
    ANDROID_GCM_API_KEY = "YOUR_FIREBASE_SERVER_KEY"

    # For newer Zulip versions (8.0+):
    # Follow official docs to set up FCM v1
    ```

3.  **Configure APNs (iOS)**:
    Add your Apple Push Notification Service certificate paths:
    ```python
    APNS_CERT_FILE = "/path/to/remote_notification_production.pem"
    APNS_KEY_FILE = "/path/to/remote_notification_production_key.pem" # If separate
    ```

4.  **Restart Zulip Server**:
    ```bash
    /home/zulip/deployments/current/scripts/restart-server
    ```

---

## Flutter Implementation

### Dependencies

Add the following packages to your `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.1.0
  shared_preferences: ^2.2.0
  firebase_messaging: ^14.7.0 # For Push Notifications
  flutter_local_notifications: ^16.0.0
```

### ZulipClient Class

Create a file named `zulip_client.dart`. This class handles all communication with the Zulip server.

```dart
import 'dart:async';
import 'dart:convert';
import 'package:http/http.dart' as http;

class ZulipClient {
  final String baseUrl;
  String? _email;
  String? _apiKey;
  String? _queueId;
  int? _lastEventId;

  // Stream controller to broadcast events to the UI
  final _eventController = StreamController<Map<String, dynamic>>.broadcast();
  Stream<Map<String, dynamic>> get eventStream => _eventController.stream;

  ZulipClient({required this.baseUrl});

  // 1. Authentication
  Future<void> authenticate(String email, String password) async {
    final uri = Uri.parse('$baseUrl/api/v1/fetch_api_key');
    final response = await http.post(uri, body: {
      'username': email,
      'password': password,
    });

    if (response.statusCode == 200) {
      final data = json.decode(response.body);
      _email = data['email'];
      _apiKey = data['api_key'];
      print('Authenticated as: $_email');
    } else {
      throw Exception('Failed to authenticate: ${response.body}');
    }
  }

  // Helper for authenticated requests
  Future<http.Response> _post(String endpoint, Map<String, dynamic> body) {
    if (_email == null || _apiKey == null) throw Exception('Not authenticated');
    final uri = Uri.parse('$baseUrl/api/v1/$endpoint');
    return http.post(uri, body: body, headers: {
      'Authorization': 'Basic ' + base64Encode(utf8.encode('$_email:$_apiKey')),
    });
  }

  Future<http.Response> _get(String endpoint, [Map<String, String>? params]) {
    if (_email == null || _apiKey == null) throw Exception('Not authenticated');
    var uri = Uri.parse('$baseUrl/api/v1/$endpoint');
    if (params != null) {
      uri = uri.replace(queryParameters: params);
    }
    return http.get(uri, headers: {
      'Authorization': 'Basic ' + base64Encode(utf8.encode('$_email:$_apiKey')),
    });
  }

  // 2. Real-time Event Queue Registration
  Future<void> registerEventQueue() async {
    final response = await _post('register', {
      'event_types': json.encode(['message', 'heartbeat', 'realm_emoji', 'reaction', 'typing']),
      'fetch_event_types': json.encode(['message']), // Fetch initial messages if needed
      'apply_markdown': 'true',
    });

    if (response.statusCode == 200) {
      final data = json.decode(response.body);
      _queueId = data['queue_id'];
      _lastEventId = data['last_event_id']; // Usually -1
      print('Queue registered: $_queueId');

      // Start the event loop
      _startEventLoop();
    } else {
      throw Exception('Failed to register queue: ${response.body}');
    }
  }

  // 3. The Event Loop (Heartbeat & Updates)
  void _startEventLoop() async {
    if (_queueId == null) return;

    while (true) {
      try {
        final uri = Uri.parse('$baseUrl/api/v1/events');
        final params = {
          'queue_id': _queueId!,
          'last_event_id': _lastEventId.toString(),
          'dont_block': 'false', // Long polling: wait for events
        };

        // Custom GET because of long timeout
        final response = await http.get(
          uri.replace(queryParameters: params),
          headers: {
            'Authorization': 'Basic ' + base64Encode(utf8.encode('$_email:$_apiKey')),
          },
        ).timeout(const Duration(seconds: 90)); // Zulip typically holds for 60s

        if (response.statusCode == 200) {
          final data = json.decode(response.body);
          final events = data['events'] as List;

          for (var event in events) {
            _handleEvent(event);
            // Update last_event_id to acknowledge receipt
            if (event.containsKey('id')) {
              int eventId = event['id'];
              if (eventId > (_lastEventId ?? -1)) {
                _lastEventId = eventId;
              }
            }
          }
        } else {
          // Handle error (e.g., queue expired, re-register)
          print('Event loop error: ${response.body}');
          await Future.delayed(Duration(seconds: 5));
          await registerEventQueue(); // Re-register
        }
      } catch (e) {
        print('Event loop exception: $e');
        await Future.delayed(Duration(seconds: 5));
      }
    }
  }

  void _handleEvent(Map<String, dynamic> event) {
    final type = event['type'];
    if (type == 'heartbeat') {
      print('Heartbeat received');
      // Connection is alive
    } else if (type == 'message') {
      print('New message: ${event['message']['content']}');
      _eventController.add(event); // Broadcast to UI
    }
    // Handle other types...
  }

  // 4. Sending Messages
  Future<void> sendMessage({
    required String type, // 'stream' or 'private'
    required dynamic to, // Stream ID (int) or List<int> user IDs
    required String content,
    required String topic, // Required for stream messages
  }) async {
    await _post('messages', {
      'type': type,
      'to': type == 'private' ? json.encode(to) : to.toString(),
      'content': content,
      'topic': topic,
    });
  }

  // 5. Register Push Notification Token
  Future<void> registerFcmToken(String token) async {
    await _post('users/me/android_gcm_reg_id', {
      'token': token,
    });
    print('FCM Token registered');
  }

  Future<void> registerApnsToken(String token, String appId) async {
    await _post('users/me/apns_device_token', {
      'token': token,
      'appid': appId,
    });
    print('APNs Token registered');
  }
}
```

---

## Real-time Events & Heartbeats

The `_startEventLoop` method in the `ZulipClient` handles the "heartbeat" of the application.
1.  **Long Polling**: The client requests `/events`. If no new events exist, the server holds the connection open (up to 60s).
2.  **Heartbeat Event**: If no real events occur during the hold time, the server sends a `heartbeat` event. This confirms the connection is alive.
3.  **Processing**: The client updates `last_event_id` and immediately sends the next request.

**Handling Disconnects**:
If the request fails or the queue expires (error `BAD_EVENT_QUEUE_ID`), the client should automatically call `registerEventQueue()` again to get a new queue ID.

---

## Push Notifications

Push notifications ensure users receive messages when the app is in the background.

1.  **Initialize Firebase** in your `main.dart`:
    ```dart
    await Firebase.initializeApp();
    FirebaseMessaging messaging = FirebaseMessaging.instance;
    NotificationSettings settings = await messaging.requestPermission();
    ```

2.  **Get the Token and Register with Zulip**:
    ```dart
    String? token = await messaging.getToken();
    if (token != null) {
      await zulipClient.registerFcmToken(token);
    }

    // Listen for token refreshes
    messaging.onTokenRefresh.listen((newToken) {
      zulipClient.registerFcmToken(newToken);
    });
    ```

3.  **Handle Incoming Messages**:
    Configure `FirebaseMessaging.onBackgroundMessage` to handle the data payload sent by Zulip. Zulip sends the message content in the `data` field.

---

## UI Integration

To integrate chat windows:

1.  **Initialize Client**: Create an instance of `ZulipClient`.
2.  **Authenticate**: Call `authenticate()`.
3.  **Connect**: Call `registerEventQueue()`.
4.  **Listen**: Use a `StreamBuilder` to listen to `client.eventStream`.

**Sample Chat Screen Structure:**

```dart
class ChatScreen extends StatefulWidget {
  final ZulipClient client;
  // ...
}

class _ChatScreenState extends State<ChatScreen> {
  final List<dynamic> _messages = [];

  @override
  void initState() {
    super.initState();
    // Listen for real-time messages
    widget.client.eventStream.listen((event) {
      if (event['type'] == 'message') {
        setState(() {
          _messages.add(event['message']);
        });
      }
    });
  }

  void _sendMessage(String text) {
    widget.client.sendMessage(
      type: 'stream',
      to: 123, // Stream ID
      topic: 'general',
      content: text,
    );
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Expanded(
          child: ListView.builder(
            itemCount: _messages.length,
            itemBuilder: (context, index) {
              final msg = _messages[index];
              return ListTile(
                title: Text(msg['sender_full_name']),
                subtitle: Text(msg['content']),
              );
            },
          ),
        ),
        // Input field and send button calling _sendMessage
      ],
    );
  }
}
```
