# Zulip Full Integration Guide for Flutter Developers

This guide provides a comprehensive walkthrough for integrating a custom Flutter application with a self-hosted Zulip server. It covers everything from basic authentication to advanced features like typing indicators, presence, file uploads, push notifications, and extensive moderation capabilities.

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
    -   [Replies & Threading](#replies--threading)
    -   [Voice Messages & Media](#voice-messages--media)
    -   [Mentions & Silent Messages](#mentions--silent-messages)
    -   [Emoji Reactions](#emoji-reactions)
    -   [Link Previews](#link-previews)
    -   [Edit & Delete Messages](#edit--delete-messages)
    -   [Private & Group Chats](#private--group-chats)
    -   [Large Public Groups](#large-public-groups)
    -   [AI Chatbot Integration](#ai-chatbot-integration)
    -   [Notifications](#notifications)
    -   [Threads (Topics)](#threads-topics)
    -   [Read Receipts & Unread Counts](#read-receipts--unread-counts)
    -   [Search (Message & Conversation)](#search-message--conversation)
    -   [Persistent Message History](#persistent-message-history)
    -   [Typing Indicators](#typing-indicators)
    -   [Message Translation](#message-translation)
7.  [Moderation & Security](#moderation--security)
    -   [Mute, Ban, Block](#mute-ban-block)
    -   [Flagging & Reporting](#flagging--reporting)
    -   [Profanity & Spam Protection](#profanity--spam-protection)
    -   [Domain Filters](#domain-filters)
8.  [Extra Utilities](#extra-utilities)
    -   [Location Sharing](#location-sharing)
    -   [Presence Indicators](#presence-indicators)
    -   [Custom Message Actions](#custom-message-actions)
    -   [Analytics](#analytics)
9.  [Push Notifications Setup](#push-notifications-setup)
10. [UI Integration](#ui-integration)

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
  flutter_html: ^3.0.0 # For rendering message content (optional)
  url_launcher: ^6.1.11 # For link handling
  # Optional: For caching/persistence
  sqflite: ^2.3.0
  path: ^1.9.0
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
  Map<String, dynamic>? _initialData;

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
    // We register for many event types.
    // fetch_event_types: ['message', 'presence'] gets initial snapshots of messages and user status.
    final response = await _post('register', {
      'event_types': json.encode([
        'message', 'heartbeat', 'realm_emoji', 'reaction', 'typing',
        'presence', 'update_message_flags', 'stream', 'subscription', 'alert_words',
        'update_message', 'delete_message', 'submessage'
      ]),
      'fetch_event_types': json.encode(['message', 'presence', 'subscription']),
      'apply_markdown': 'true',
      'client_gravatar': 'true',
    });

    if (response.statusCode == 200) {
      final data = json.decode(response.body);
      _queueId = data['queue_id'];
      _lastEventId = data['last_event_id']; // Usually -1
      _initialData = data; // Store initial snapshot (presences, unread counts, etc.)

      // Handle initial presence snapshot immediately if needed
      if (data.containsKey('presences')) {
         _eventController.add({'type': 'initial_presence', 'presences': data['presences']});
      }

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
        } else if (response.statusCode == 429) {
           // Rate Limit Handling
           print('Rate limited. Waiting...');
           await Future.delayed(Duration(seconds: 10));
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
```

---

## Advanced Features

### Replies & Threading
Zulip threading is topic-based. To reply, send a message to the same Stream ID and Topic.
To quote:
```markdown
@**User** [said](link):
> Quote
Reply
```

### Voice Messages & Media
Upload the file using `uploadFile()`, then send a message with the link:
```dart
String uri = await client.uploadFile(path);
await client.sendMessage(..., content: '[Voice Note]($uri)');
// Video/Image: ![Title]($uri)
```

### Mentions & Silent Messages
-   **Mention**: `@**User Name**`
-   **Silent Mention**: `@_**User Name**` (No push notification)
-   **Global**: `@**all**` (use sparingly)

### Emoji Reactions
```dart
await client.addReaction(messageId, 'thumbs_up');
```

### Link Previews
Render the `content` (HTML) from the message object using `flutter_html`. Zulip servers generate Open Graph previews in the HTML automatically.

### Edit & Delete Messages
```dart
await client.updateMessage(msgId, 'New text');
await client.deleteMessage(msgId);
```

### Private & Group Chats
Private 1-1 and Group chats use the same `private` type.
```dart
// Group Chat
await client.sendMessage(
  type: 'private',
  to: [101, 102, 103],
  content: 'Team update',
  topic: '',
);
```

### Large Public Groups
Use **Streams**. Streams can have hundreds or thousands of subscribers. Use the `stream` type in `sendMessage`.

### AI Chatbot Integration
1.  **Create Bot**: Create a "Generic Bot" in Zulip settings.
2.  **Webhook**: Set up an Outgoing Webhook pointing to your AI service.
3.  **Process**: When the bot is mentioned, Zulip POSTs to your service. Your service replies to the webhook to post the AI response.

### Notifications
Handled via the Event Queue (foreground) and FCM/APNs (background). See "Push Notifications Setup".

### Threads (Topics)
Topics are just string identifiers in a stream. You can create a new thread simply by sending a message with a new topic name.

### Read Receipts & Unread Counts
-   **Read Receipts**: Call `markAsRead([ids])`.
-   **Unread Counts**: Track `update_message_flags` events. If `op: 'add', flag: 'read'`, decrement count. If `op: 'remove', flag: 'read'`, increment.

### Search (Message & Conversation)
Use `getMessages` with `narrow`.
**Message Search**:
```dart
narrow: [{'operator': 'search', 'operand': 'query'}]
```
**Conversation Search**:
```dart
narrow: [{'operator': 'stream', 'operand': 'general'}, {'operator': 'search', 'operand': 'query'}]
```

### Persistent Message History
To persist history offline:
1.  Use `sqflite` or `hive`.
2.  On app start, load from DB.
3.  Call `getMessages(anchor: lastDbId, ...)` to fetch new messages.
4.  Handle `delete_message` and `update_message` events to update your local DB.

### Typing Indicators
Use `sendTypingStatus(op: 'start'/'stop')`. Listen for `typing` events to show UI indicators.

### Message Translation
Client-side implementation:
1.  Long-press message.
2.  Call Google Translate API.
3.  Show translation overlay.
(Zulip has no native translation API).

---

## Moderation & Security

### Mute, Ban, Block
-   **Mute User**: User-side preference. Store a list of muted User IDs locally and filter them out of your UI list.
-   **Mute Topic**: Use `/users/me/subscriptions/muted_topics`.
-   **Ban User**: (Admin only) Use `/users/{user_id}` PATCH to deactivate a user.
-   **Block**: Zulip doesn't have a "block" feature exactly like social media; "Mute" is the standard.

### Flagging & Reporting
Implement a UI action "Report". Since there's no native "Report" API, this usually means sending a structured message to an Admin stream or via email.

### Profanity & Spam Protection
-   **Profanity**: Implement client-side filtering or use a server-side outgoing webhook to analyze/delete messages.
-   **Spam**: The `ZulipClient` handles 429 Rate Limit errors by waiting.
-   **Domain Filters**: Configure "Restrict to domain" in Server Settings (Admin panel) to prevent unauthorized signups.

---

## Extra Utilities

### Location Sharing
Send a Geo URI or Maps Link: `https://maps.google.com/?q=lat,long`. Zulip renders the link.

### Presence Indicators
Poll or listen to `presence` events.
-   **Home Screen**: Use the initial snapshot from `register()`.
-   **Real-time**: Listen to the event stream for updates.

### Custom Message Actions
Use `submessage` events for app-specific data (like a poll or game move) that shouldn't render as text.
Or, use raw JSON in the content (e.g., `json { "poll_id": 123 } `) and parse it client-side if it matches a specific schema.

### Analytics
Fetch server stats (Admin): `GET /api/v1/analytics/website`.

---

## Push Notifications Setup

1.  **Firebase**: Add `firebase_messaging`.
2.  **Register**:
    ```dart
    String? token = await FirebaseMessaging.instance.getToken();
    await client.registerFcmToken(token!);
    ```
3.  **Background**: Define a top-level function `Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message)`.

---

## UI Integration

To integrate chat windows:
1.  **Initialize Client**: `ZulipClient(baseUrl: ...)`
2.  **Authenticate**: `client.authenticate(...)`
3.  **Start**: `client.start()`
4.  **UI**: Wrap your Chat Screen in a `StreamBuilder(stream: client.eventStream)`.

**Pro Tip**: Use a `Provider` or `GetIt` to make the `ZulipClient` a singleton accessible throughout the app.
