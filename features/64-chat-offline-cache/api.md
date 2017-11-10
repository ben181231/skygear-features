# Objective

- Provide cached result to Chat Container API calls
- Compatible to old API interface

# Overview

Container contains a local store for caching resources. The local store caches resources by resource id.

### Store Update

When SDK recieves result from server, for each record, the newly fetched record would always replace the old one with the same record id if there is one.

### API

For fetch API of each resource, there would be a new callback that should return result from cache immediately.

The interface for each language may be different.

```
getResource(params, callback, cachedCallback?)
```

Resource includes conversation and message.

The cachedCallback should be optional, thus compatible to the old API interace.

### Expected UI Display / Common Use Case

Client App is expected to keep a single list of messages for display in table. And user scroll beyond the list, they would need to call API to fetch messages.

Client App should replace old message with new message if the same message id is found.

Deleted messages are also returned in fetch API, with empty content and marked as deleted. This is to keep the whole message list complete. Client App may or may not skip these messages in UI display.

The conversation view controller in UI Kit would use the cached version of chat API. Thus, developer should be able to use this new feature easily.

The controller should have the following behaviour:

- keep a cursor pointing to a message or the latest message, set a window size of how many messages should be kept in memory
- when moving the cursor around (scrolling around or jumping around), it should erase messages outside the window and fetch messages newly entered the window
- fetch the messages to fill the window when enter view, if the cursor is not set, it would stick to the latest message
- subscribe to new messages, but only append to bottom if the window is on the latest message side
- display, allow resend, cancel pending sent message, i.e. send by user but not responded by server yet
- sync latest messages on reconnect if the window is on the latest message side

# Sample Code for Conversation List

### In conversation list

#### Start subscribing conversation changes

When
- enter conversation list

```
skygearChat.subscribeConversations(completion: func (conversationDeltas) {
  skygearChat.fetchConversations(completion: func(conversations, cached = false) {
    if (cached) {
      return
    }

    this.conversations = conversations
    this.tableView.reloadTable()
  })
})
```

#### Fetch and update table

When
- enter conversation list

```
skygearChat.fetchConversations(completion: func (conversations, cached = false) {
  this.conversations = conversations
  this.tableView.reloadTable()
})
```

### In message list

#### Subscribe messages

When
- enter message list

```
skygearChat.subscribeMessages(conversation, func (messageDeltas) {
  // Merge delta to current list

  // Apply the delta to UI one by one
  // OR
  // Reload the UI once
})
```

When pubsub disconnect and reconnect, messages in between are not fetched automatically. Client app may fetch latest messages when pubsub reconnected.

```
let latestFetchedMessage: Message;

skygear.pubsub.onDisconnect(func () {
  this.latestFetchedMessage = this.messages.first // OR this.messages.last
})

skygear.pubsub.onConnect(func () {
  let afterMessage = this.latestFetchedMessage
  skygearChat.fetchMessages(conversation, afterMessage, limit, oreder, /* Update UI */)
})
```

#### Fetch messages

When
- enter message list
- load messages when scroll to top of the list

```
skygearChat.fetchMessages(conversation, beforeMessage / afterMessage, limit, order, func (messageDeltas, cached = false) {
  // Merge delta to current list

  // Apply the delta to UI one by one
  // OR
  // Reload the UI once
})
```

### Create messages

```
// Promise
skygearChat.createMessage(/* ... */, (message) => {
    // insert the new message in UI
  })
  .then((message) => {
    // update the message
  });

// Non-promise
skygearChat.createMessage(/* ... */, func (message) => {
  // check `syncingToServer`, `alreadySyncToServer` and `fail` of the message
  // to insert or update message in UI
})
```

# Changes Required

Please also see [fetch_message_after_message.md](./fetch_message_after_message.md)

### Server

#### Schema Change

Add a `previous_message` column to `message` table, to determine if two given messages are subsequent messages.

Deleted messages are also returned in fetch API, with empty content and marked as deleted. This is to keep the whole message list complete.

#### API

Use message instead of time as the query reference in fetchMessages API, this can avoid missing messages when there are multiple messages which have the same timestamp.

old:

```
fetchMessages(conversation_id, limit, before_time?, order?)
```

new:

```
fetchMessages(conversation_id, limit, before_message?, order?)
```

### SDK

- Make use of `syncingToServer`, `alreadySyncToServer` and `fail` (found in iOS but not in JS and Android) to cache and restore unsent message.

- Add `sendDate` to `message` record for sorting.

- Add cached callback to create message API.

The cachedCallback is expected to call immediately after adding to cache store.

```
func createMessage(/* content params */, cachedCallback?)

// Promise
skygearChat.createMessage(/* ... */, (message) => {
    // insert the new message in UI
  })
  .then((message) => {
    // update the message
  });

// Non-Promise
skygearChat.createMessage(/* ... */, func (message) => {
  // check `syncingToServer`, `alreadySyncToServer` and `fail` of the message
  // to insert or update message in UI
})
```

- Add cached callback to fetch messages and fetch conversations API.

```
func fetchConversations(cachedCallback?)
func fetchMessages(conversation, beforeMessage, limit, order, cachedCallback?)
```

#### JS

```js
const successCallback = (conversations, cached = false) => {
  // display conversations
};

skygearChat
  .getConversations(successCallback)
  .then(successCallback, errorCallback);

// no-cache version
skygearChat
  .getConversations()
  .then(successCallback, errorCallback);
```

```js
let currentMessages = [];

const successCallback = (messages, cached = false) => {
  currentMessages = skygearChat.merge(currentMessages, messages);
  // display currentMessages
};

skygearChat
  .getMessages(conversation, beforeMessage, limit, order, successCallback)
  .then(successCallback, errorCallback);

// no-cache version
skygearChat
  .getMessages(conversation, beforeMessage, limit, order)
  .then(successCallback, errorCallback);
```

#### Non-JS

```
skygearChat.fetchConversations(func (conversations, cached = false) {
  // display conversations
})
```

```
let currentMessages = [];

skygearChat.fetchMessages(func (messages, cached = false) {
  currentMessages = skygearChat.merge(currentMessages, messages);
  // display currentMessages
})
```

- Add cache store to skygear chat

There should be a common cache logic layer and cache implementation.

#### Cache implementation

Skygear may provide cache implementation for each platform, and provide functions / classes to integrate with the app existing persistant store.

- iOS: Core Data
- Android: SQLite
- JS: localforage
- iOS + Android: Realm

```
func set(conversation: Conversation, forID: String) -> Void
func set(conversation: Conversation, message: Message, forID: String) -> Void
func purgeAll() -> Void

// if not implementated, the common cache logic needs to get all and filter the messages
func fetchMessage(conversation: Conversation, before/afterMessageID: String, limit: Int, order: String) -> [Message]
```

#### Common cache logic

The cache logic needs to implement the interface of the API that support cacheCallback or update the cache.

For security reason, the cache should live with a user session only. When a user logout, the cache should be erased.

interface for converation list

```
func saveConversation(conversation: Conversation) -> Void
func deleteConversation(conversation: Conversation) -> Void
func fetchConversations() -> [Conversation]
func leaveConversation(conversation: Conversation) -> Void
```

interface for message list

```
func createMessage(conversation: Conversation, body: ...) -> Void
func editMessage(conversation: Conversation, body: ...) -> Void
func deleteMessage(conversation: Conversation, message: Message) -> Void
func fetchMessage(conversation: Conversation, before: Message, limit: Int, order: String) -> [Message]
func markDeliveredMessages(messages: [Message])
func fetchReceiptsWithMessage(message: Message)
```