# Zulip Extra Features Guide for Flutter Developers

This guide documents how to implement advanced features in your custom Flutter Zulip application, building upon the foundation laid in the [Integration Guide](./ZULIP_INTEGRATION_GUIDE.md).

## Table of Contents
1. [Typing Indicators](#typing-indicators)
2. [Presence (Online Status)](#presence-online-status)
3. [File & Voice Message Uploads](#file--voice-message-uploads)
4. [Replies & Threading](#replies--threading)
5. [Emoji Reactions](#emoji-reactions)
6. [Read Receipts](#read-receipts)

---

## Typing Indicators

To show "User is typing..." in the UI, you need to handle the `typing` event type and send typing notifications when your user is typing.

### sending Typing Status

Add this method to your `ZulipClient`:

```dart
  // Send typing status
  // op: 'start' or 'stop'
  Future<void> sendTypingStatus({
    required String op,
    required List<int> to, // User IDs for private messages
    // OR
    // required int streamId,
    // required String topic,
  }) async {
    // Example for Private Message typing status
    await _post('typing', {
      'type': 'private',
      'op': op,
      'to': json.encode(to),
    });

    // For Stream typing status:
    // 'type': 'stream',
    // 'to': streamId.toString(),
    // 'topic': topic,
  }
```

**Usage:** Call `sendTypingStatus(op: 'start', ...)` when the text field changes. Use a debouncer to call `stop` after a few seconds of inactivity.

### Receiving Typing Events

In your event loop listener:

```dart
    if (type == 'typing') {
      final senderId = event['sender']['user_id'];
      final op = event['op']; // 'start' or 'stop'
      print('User $senderId is ${op} typing');
      // Update UI state
    }
```

---

## Presence (Online Status)

Presence indicates if a user is "active", "idle", or "offline". This information can be shown on the home screen or chat list.

### Updating Your Presence

Zulip clients regularly report their status to the server (e.g., every minute).

```dart
  Future<void> updatePresence(String status) async {
    // status: 'active' or 'idle'
    await _post('users/me/presence', {
      'status': status,
      'ping_only': 'false', // 'true' if just keeping connection alive without status change
    });
  }
```

### Fetching Presence

You can fetch the presence of all users or specific users. Note: The initial `register` call often includes a snapshot of presence if `fetch_event_types` includes `presence`.

```dart
  Future<Map<String, dynamic>> getUserPresence(int userId) async {
    final response = await _get('users/$userId/presence');
    if (response.statusCode == 200) {
      return json.decode(response.body);
    }
    throw Exception('Failed to get presence');
  }
```

---

## File & Voice Message Uploads

Zulip treats voice messages as regular file uploads.

### Uploading a File

This requires a `MultipartRequest`. Add this to `ZulipClient`:

```dart
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
      return data['uri']; // Returns the Zulip internal URI (e.g., /user_uploads/...)
    } else {
      throw Exception('Upload failed: $respStr');
    }
  }
```

### Sending a Voice Message

1.  **Record Audio**: Use a Flutter package like `flutter_sound` or `record` to save audio to a local file (e.g., `recording.m4a`).
2.  **Upload**: Call `uploadFile('path/to/recording.m4a')`.
3.  **Send Message**: Send a message with Markdown syntax linking to the uploaded file. Zulip clients render audio files with a player if the syntax is correct.

```dart
  void sendVoiceMessage(String localPath) async {
    final uploadedUri = await client.uploadFile(localPath);
    // Construct Markdown link
    final content = '[Voice Message]($uploadedUri)';

    await client.sendMessage(
      type: 'private',
      to: [123],
      content: content,
      topic: '',
    );
  }
```

---

## Replies & Threading

Zulip's "reply" model is different from WhatsApp or Slack.

-   **Streams**: Public/Private channels.
-   **Topics**: Subjects within a stream.

**"Replying" to a message in a Stream:**
You simply send a new message to the **same Stream ID** and **same Topic name**.

```dart
  client.sendMessage(
    type: 'stream',
    to: streamId,
    topic: 'Sales Meeting', // Must match the original message's topic
    content: 'Yes, I agree.',
  );
```

**Quote & Reply:**
To visually quote a message (like in WhatsApp), you include the quote in the Markdown content:

```markdown
@**User Name** [said](link_to_msg):
> This is the quoted text.

Here is my reply.
```

---

## Emoji Reactions

Users can react to messages with emojis.

```dart
  Future<void> addReaction(int messageId, String emojiName) async {
    // emojiName: e.g., 'thumbs_up', 'heart'
    await _post('messages/$messageId/reactions', {
      'emoji_name': emojiName,
      // For standard emojis, emoji_code is usually the same or hex code
    });
  }

  Future<void> removeReaction(int messageId, String emojiName) async {
    await _delete('messages/$messageId/reactions', {
      'emoji_name': emojiName,
    });
  }
```

**Handling Reaction Events:**
Listen for `type: reaction` in the event loop to update the UI (increment/decrement reaction counts).

---

## Read Receipts

To mark messages as read:

```dart
  Future<void> markAsRead(List<int> messageIds) async {
    await _post('messages/flags', {
      'messages': json.encode(messageIds),
      'op': 'add',
      'flag': 'read',
    });
  }
```

**Note**: Zulip automatically marks messages as read if you fetch them with certain flags, but explicitly marking them is safer for custom UIs.
