---
layout: post
title: "Designing a WhatsApp System"
---

This is my attempt at thinking through a WhatsApp-like system.

I liked this one because it is not just a normal CRUD-style system. Once I started thinking through it, there were a few things that immediately made it feel different:

* real-time messaging
* offline delivery
* stateful connections
* very large scale

These are basically my notes on how I’d reason through it right now.

---

## Requirements

### Non-functional

* Availability > Consistency
* Low latency for send and receive
* Handle very large scale
* Ensure messages are eventually delivered

---

## Core Entities

The main pieces I care about are:

* user
* chat
* message
* attachment

---

## API Style

This one feels more event-driven than normal REST because the system needs to push updates back to the client in real time.

### Client → Server

* createChat
* sendMessage
* updateChat
* sendAttachment

### Server → Client

* chatCreated
* chatUpdated
* messageReceived
* newAttachment

---

## High-level architecture
<img src="{{ '/images/whatsapp-arch.png' | relative_url }}" alt="whatsapp system architecture diagram">

I split this into a few main pieces:

- Chat service → manages chats + participants  
- Message service → handles sending messages  
- Database → stores chats + messages  
- Async worker → handles fan-out (writing to inbox)  

When a message is sent:
- it gets stored first  
- then an async worker figures out who should receive it  

This keeps the write path fast and pushes the heavier work to background processing.

---

## Creating Chats

When a user creates a chat, the basic flow in my head is:

1. request goes through the load balancer to the Chat Service
2. Chat Service creates a new `chatId`
3. write chat metadata
4. write chat participants

I would probably keep a `ChatParticipants` table because one of the main queries I care about is:

> what chats does this user belong to?

So I’d structure it around:

* partition key: `userId`
* sort key: `chatId`

That makes it easy to fetch all chats for a given user.

Note: I decided to use dynamoDB here as it handles massive throughput, horizontal scaling, and is managed so it ensures resiliency and operational simplicity

---

## Message Storage

For message storage, I’d create a `Message` table with:

* partition key: `chatId`
* sort key: `createdAt`

Other useful fields:

* `messageId`
* `content`
* `attachment`
* `senderId`

The reason this structure makes sense to me is that the main read pattern is:

> give me messages for this chat

So grouping by `chatId` and sorting by creation time feels natural.

One thing I haven’t fully solved here is strict message ordering.

Using `createdAt` as a sort key gives a reasonable ordering within a chat, but in a real system we might need something stronger like sequence numbers or server-side ordering guarantees.

---

## Reading Messages

When the client opens a chat:

* it already knows the `chatId`
* it requests messages for that chat
* the backend queries by `chatId`
* results are paginated by `createdAt`

That part feels pretty standard compared to the rest of the system.

---

## Real-Time Delivery

At first I thought:

> just poll the server

That falls apart pretty fast at this scale.

So I’d use **WebSockets** instead.

The mental model for me is:

* client opens a persistent connection
* server keeps that connection alive
* new messages get pushed down immediately

That avoids constant polling and reduces a lot of unnecessary traffic.

---

## Handling Offline Users

This was one of the first parts that made the system feel more interesting.

If the user is offline, I still need the message to show up later.

The way I’d think about that is with an inbox-like table.

### Flow

1. Message Service writes the message to the `Message` table
2. it emits an event
3. an async worker consumes that event
4. the worker finds the participants in the chat
5. for any offline user, it writes a pending entry into that user’s inbox

So the inbox table would look something like:

* `userId`
* `messageId`
* `chatId`

---

## When the User Reconnects

When the client reconnects:

* the server checks the inbox
* delivers any pending messages
* once the client acknowledges them, those inbox entries can be removed

That way the inbox is really just there as a delivery backlog, not the primary source of truth.

---

## When the User Is Already Online

If the user already has an active WebSocket connection:

* send the message directly
* skip the inbox path unless delivery fails

So in my head the inbox is really for:

* offline users
* fallback when direct delivery does not succeed

---

## Sending Attachments

For attachments, I would not store the actual file directly in the main database.

The flow I’d expect is:

1. user uploads an attachment
2. Message Service stores the file in blob storage like S3
3. the service gets back a URL or object reference
4. the message record stores that reference
5. clients download the attachment from blob storage

That keeps the message path lighter and avoids pushing large binary objects into the message database.

---

## Scaling the System

At this scale, most of the services need to be horizontally scalable.

The parts that scale nicely are:

* stateless service layer
* async workers
* storage systems that are already designed for partitioning

The part that gets trickier is WebSockets, because WebSockets make the connection layer stateful.

---

## Why WebSockets Make This Harder

This is where the design stopped feeling straightforward to me.

If:

* User A is connected to Server 1
* User B is connected to Server 2

and B sends a message to A, then Server 2 needs some way to get that message over to Server 1.

That is where I started thinking about Pub/Sub.

---

## Pub/Sub for Cross-Server Routing

The way I think about it is:

* each WebSocket server knows which users are connected to it
* each server subscribes to the users it is currently responsible for

So if User B sends a message to User A:

1. Server 2 receives the message
2. Server 2 publishes an event for User A
3. Pub/Sub routes that event to Server 1
4. Server 1 pushes the message to User A over the active WebSocket connection

The simple mental model for me is:

* Pub/Sub handles server-to-server routing
* WebSocket handles final delivery to the client

---

## Handling Reconnection

When a user disconnects:

* remove the active connection from the connection registry
* unsubscribe or update the server’s routing state

When the user reconnects:

* the load balancer routes them to some server
* that server registers the connection
* checks the inbox for missed messages
* delivers whatever is still pending

---

## Handling Multiple Devices

A user may be connected on more than one device:

* phone
* laptop
* tablet

The simplest way for me to think about that is to treat each device as its own active connection.

So:

* one `userId` can map to multiple active connections
* when a message arrives, it gets pushed to all active devices for that user

That keeps the user experience consistent across devices.

---

## Tradeoffs

This design gives me:

* real-time delivery
* offline delivery
* horizontal scalability for most components

But the tradeoffs are:

* eventual consistency
* stateful WebSocket infrastructure
* extra complexity from Pub/Sub
* extra writes for offline inbox handling

For this kind of system, those tradeoffs feel worth it.

---

## What I Learned

The biggest things this made me realize were:

* real-time systems force you to think a lot more about connection state
* delivery guarantees usually mean extra storage somewhere
* Pub/Sub becomes really useful once users are connected to different servers
* the hard part is not just storing the data, it is routing and delivering it correctly

---

## Final Thoughts

This one helped me realize that real-time systems need a pretty different mental model than normal request/response systems.

There is a lot more coordination involved:

* connection state
* delivery guarantees
* cross-server routing

I’m still learning this area, but this one was a really useful problem to work through.

---

## Future Improvements

* stronger message ordering guarantees
* read receipts
* end-to-end encryption
* multi-region delivery
* per-user rate limiting
