# Zulip Full Integration Guide for Flutter Developers

This guide provides a comprehensive walkthrough for integrating a custom Flutter application with a self-hosted Zulip server. It covers everything from basic authentication to advanced features like typing indicators, presence, file uploads, and push notifications.

## Table of Contents
1.  [Overview](#overview)
2.  [Prerequisites](#prerequisites)
3.  [Server Configuration](#server-configuration)
4.  [Flutter Implementation](#flutter-implementation)
    -   [Dependencies](#dependencies)
    -   [ZulipClient Class (Complete)](#zulipclient-class-complete)
5.  [Core Usage](#core-usage)
    -   [Authentication](#authentication)
    -   [Starting the Event Loop](#starting-the-event-loop)
    -   [Sending Messages](#sending-messages)
6.  [Advanced Features](#advanced-features)
    -   [Typing Indicators](#typing-indicators)
    -   [Presence (Online Status)](#presence-online-status)
    -   [File & Voice Message Uploads](#file--voice-message-uploads)
    -   [Replies & Threading](#replies--threading)
    -   [Emoji Reactions](#emoji-reactions)
    -   [Read Receipts](#read-receipts)
    -   [Fetching Message History](#fetching-message-history)
    -   [Editing & Deleting Messages](#editing--deleting-messages)
    -   [Mentions](#mentions)
    -   [Unread Message Counts](#unread-message-counts)
    -   [Group Chats (Private Groups)](#group-chats-private-groups)
    -   [Search & Filtering](#search--filtering)
7.  [Push Notifications](#push-notifications)
8.  [UI Integration](#ui-integration)

---

## Overview

Zulip is a real-time chat application with a unique threading model. Unlike other chat apps, Zulip organizes conversations into **Streams** (like channels) and **Topics** (sub-threads within streams).

Integration relies on a few key concepts:
-   **API Key**: Used for authentication.
-   **Event Queue**: A long-polling mechanism to receive real-time updates (messages, reactions, typing status, heartbeats).
-   **Push Notifications**: Handled via FCM (Android) or APNs (iOS).

## Prerequisites

-   A **self-hosted Zulip server** (version 4.0+ recommended).
-   A **Flutter project**.
-   Access to the server's `/etc/zulip/settings.py` for push notification configuration.

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

### ZulipClient Class (Complete)

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
  bool _isEventLoopRunning = false;

  // Stream controller to broadcast events to the UI
  final _eventController = StreamController<Map<String, dynamic>>.broadcast();
  Stream<Map<String, dynamic>> get eventStream => _eventController.stream;

  ZulipClient({required this.baseUrl});

  // --- Core Methods ---

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

  // 2. Start Listening for Events
  void start() {
    if (_isEventLoopRunning) return;
    _isEventLoopRunning = true;
    _startEventLoop();
  }

  void stop() {
    _isEventLoopRunning = false;
  }

  Future<void> _registerEventQueue() async {
    final response = await _post('register', {
      'event_types': json.encode(['message', 'heartbeat', 'realm_emoji', 'reaction', 'typing', 'presence', 'update_message_flags']),
      'fetch_event_types': json.encode(['message']), // Fetch initial messages if needed
      'apply_markdown': 'true',
    });

    if (response.statusCode == 200) {
      final data = json.decode(response.body);
      _queueId = data['queue_id'];
      _lastEventId = data['last_event_id']; // Usually -1
      print('Queue registered: $_queueId');
    } else {
      throw Exception('Failed to register queue: ${response.body}');
    }
  }

  // 3. The Event Loop (Heartbeat & Updates)
  void _startEventLoop() async {
    while (_isEventLoopRunning) {
      try {
        // If we don't have a queue ID, register one first
        if (_queueId == null) {
          try {
            await _registerEventQueue();
          } catch (e) {
            print('Registration failed: $e');
            await Future.delayed(Duration(seconds: 5));
            continue; // Retry loop
          }
        }

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
          // If queue is invalid (BAD_EVENT_QUEUE_ID), reset ID to trigger re-registration
          _queueId = null;
          await Future.delayed(Duration(seconds: 2));
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
    } else {
       // Forward other events (typing, reactions, etc.)
       _eventController.add(event);
    }
  }

  // --- HTTP Helpers ---

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

  Future<http.Response> _patch(String endpoint, Map<String, dynamic> body) {
    if (_email == null || _apiKey == null) throw Exception('Not authenticated');
    final uri = Uri.parse('$baseUrl/api/v1/$endpoint');
    return http.patch(uri, body: body, headers: {
      'Authorization': 'Basic ' + base64Encode(utf8.encode('$_email:$_apiKey')),
    });
  }

  Future<http.Response> _delete(String endpoint, [Map<String, String>? params]) {
    if (_email == null || _apiKey == null) throw Exception('Not authenticated');
    var uri = Uri.parse('$baseUrl/api/v1/$endpoint');
    if (params != null) {
      uri = uri.replace(queryParameters: params);
    }
    return http.delete(uri, headers: {
      'Authorization': 'Basic ' + base64Encode(utf8.encode('$_email:$_apiKey')),
    });
  }

  // --- Feature Methods ---

  // Sending Messages
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

  // Push Notifications
  Future<void> registerFcmToken(String token) async {
    await _post('users/me/android_gcm_reg_id', {'token': token});
    print('FCM Token registered');
  }

  Future<void> registerApnsToken(String token, String appId) async {
    await _post('users/me/apns_device_token', {'token': token, 'appid': appId});
    print('APNs Token registered');
  }

  // Typing Indicators
  Future<void> sendTypingStatus({
    required String op, // 'start' or 'stop'
    required List<int> to, // User IDs for private messages
  }) async {
    await _post('typing', {
      'type': 'private',
      'op': op,
      'to': json.encode(to),
    });
  }

  // Presence
  Future<void> updatePresence(String status) async {
    await _post('users/me/presence', {
      'status': status,
      'ping_only': 'false',
    });
  }

  Future<Map<String, dynamic>> getUserPresence(int userId) async {
    final response = await _get('users/$userId/presence');
    if (response.statusCode == 200) return json.decode(response.body);
    throw Exception('Failed to get presence');
  }

  // File Uploads
  Future<String> uploadFile(String filePath) async {
    if (_email == null || _apiKey == null) throw Exception('Not authenticated');

    final uri = Uri.parse('$baseUrl/api/v1/user_uploads');
    final request = http.MultipartRequest('POST', uri)
      ..headers['Authorization'] = 'Basic ' + base64Encode(utf8.encode('$_email:$_apiKey'))
      ..files.add(await http.MultipartFile.fromPath('file', filePath));

    final response = await request.send();
    final respStr = await response.stream.bytesToString();

    if (response.statusCode == 200) {
      final data = json.decode(respStr);
      return data['uri'];
    } else {
      throw Exception('Upload failed: $respStr');
    }
  }

  // Reactions
  Future<void> addReaction(int messageId, String emojiName) async {
    await _post('messages/$messageId/reactions', {'emoji_name': emojiName});
  }

  Future<void> removeReaction(int messageId, String emojiName) async {
    await _delete('messages/$messageId/reactions', {'emoji_name': emojiName});
  }

  // Read Receipts
  Future<void> markAsRead(List<int> messageIds) async {
    await _post('messages/flags', {
      'messages': json.encode(messageIds),
      'op': 'add',
      'flag': 'read',
    });
  }

  // Fetch History / Search
  Future<List<dynamic>> getMessages({
    required int anchor,
    required int numBefore,
    required int numAfter,
    required List<Map<String, String>> narrow,
  }) async {
    final response = await _get('messages', {
      'anchor': anchor.toString(),
      'num_before': numBefore.toString(),
      'num_after': numAfter.toString(),
      'narrow': json.encode(narrow),
      'apply_markdown': 'true',
    });

    if (response.statusCode == 200) {
      final data = json.decode(response.body);
      return data['messages'];
    }
    throw Exception('Failed to fetch messages');
  }

  // Edit/Delete
  Future<void> updateMessage(int messageId, String newContent) async {
    await _patch('messages/$messageId', {'content': newContent});
  }

  Future<void> deleteMessage(int messageId) async {
    await _delete('messages/$messageId');
  }
}
```

---

## Core Usage

### Authentication
```dart
final client = ZulipClient(baseUrl: 'https://chat.example.com');
await client.authenticate('user@example.com', 'password123');
```

### Starting the Event Loop
Once authenticated, start listening for real-time events. This handles heartbeats automatically.
```dart
client.start();
client.eventStream.listen((event) {
  print('Received event: $event');
});
```

### Sending Messages
```dart
// Send to a Stream
await client.sendMessage(
  type: 'stream',
  to: 123, // Stream ID
  topic: 'general',
  content: 'Hello everyone!',
);

// Send to a Private User
await client.sendMessage(
  type: 'private',
  to: [456], // User ID
  topic: '',
  content: 'Hi there!',
);
```

---

## Advanced Features

### Typing Indicators
**Send Status:**
```dart
// Start typing
client.sendTypingStatus(op: 'start', to: [456]);
// Stop typing (after delay)
client.sendTypingStatus(op: 'stop', to: [456]);
```

**Receive Status:**
Listen for events where `type == 'typing'`.

### Presence (Online Status)
**Update Status:**
```dart
// Set as 'active'
client.updatePresence('active');
```

**Get Status:**
```dart
final presence = await client.getUserPresence(456);
print(presence);
```

### File & Voice Message Uploads
Voice messages are just file uploads linked in Markdown.

```dart
// 1. Record audio to a local file
String audioPath = '/path/to/audio.m4a';

// 2. Upload
String uri = await client.uploadFile(audioPath);

// 3. Send Message with Link
await client.sendMessage(
  type: 'private',
  to: [456],
  content: '[Voice Message]($uri)',
  topic: '',
);
```

### Replies & Threading
To reply in a stream, simply send a message to the **same Stream ID** and **same Topic name**.

To quote a message:
```markdown
@**User Name** [said](link_to_msg):
> Quoted text here
My reply here.
```

### Emoji Reactions
```dart
await client.addReaction(messageId, 'thumbs_up');
```

### Read Receipts
```dart
await client.markAsRead([1001, 1002]);
```

### Fetching Message History
```dart
final history = await client.getMessages(
  anchor: 0, // Oldest
  numBefore: 0,
  numAfter: 50,
  narrow: [{'operator': 'stream', 'operand': 'general'}],
);
```

### Editing & Deleting Messages
```dart
await client.updateMessage(msgId, 'Corrected text');
await client.deleteMessage(msgId);
```

### Mentions
Use Markdown syntax: `@**User Name**`.
When parsing messages, look for `flags: ['mentioned']`.

### Unread Message Counts
Track local state by listening to `update_message_flags` events. Check if the `read` flag is added or removed.

### Group Chats (Private Groups)
Send to multiple user IDs:
```dart
await client.sendMessage(
  type: 'private',
  to: [123, 456, 789],
  content: 'Team update!',
  topic: '',
);
```

### Search & Filtering
Use the `narrow` parameter in `getMessages`:
```dart
// Search for "urgent"
await client.getMessages(..., narrow: [{'operator': 'search', 'operand': 'urgent'}]);
```

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
3.  **Connect**: Call `start()`.
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
