# I Was Asked to Design WhatsApp in a Senior Interview. Here's My Full Architecture Breakdown.

** By Subrata Kumar — Tech Lead, React Native **

---

The interviewer said four words: **"Design a messaging app."** Then waited.

It was a senior-level interview at a company I really wanted. I'd prepared. I'd read the usual resources. And then the interviewer said — casually, like it was nothing — "Walk me through how you'd design a messaging system. Like WhatsApp."

The problem wasn't that I didn't know anything. The problem was that I knew *about* WhatsApp. I knew what it did. What I wasn't prepared for was the moment the interviewer asked: *"Okay — if User A and User B are on different servers, how does the message get from one to the other?"*

Silence. Not the confident, thinking-out-loud kind. The kind where you feel the room change.

That question — that specific routing problem — is where most answers fall apart. This article is the full breakdown I wish I'd had before walking into that room. Not a surface-level description of WhatsApp. An actual system design answer, with the trade-offs named and the reasoning made explicit.

---

## Step 1: Start with requirements, not architecture

The biggest mistake candidates make is drawing boxes before asking questions. The first 3–5 minutes of a system design interview should be you clarifying scope. Not drawing.

**Functional Requirements (in scope):**
- 1:1 real-time messaging
- Group messaging (up to 256 members)
- Media sharing (images, video, documents)
- Online / offline status indicators
- Message delivery receipts (sent → delivered → read)

**Non-Functional Requirements:**
- 2 billion users, ~100M concurrent connections
- Message delivery under 100ms
- Zero message loss (even when receiver is offline)
- End-to-end encrypted
- 99.99% availability

> **Interview tip:** Spending 3–5 minutes here signals engineering maturity. Someone who jumps straight to Kafka and Cassandra before clarifying scale is a red flag, not a green one. Interviewers want to see that you ask before you assume.

---

## Step 2: High-level architecture

Once requirements are locked, give the 30,000-foot view before going deep. Name the major components and how they relate. This gives the interviewer a map — and signals that you won't lose the thread when you zoom in.

```
┌─────────────┐     ┌──────────────┐     ┌──────────────────┐     ┌───────────┐
│  iOS/Android│────▶│ Load Balancer│────▶│  Chat Server 1   │────▶│   Kafka   │
└─────────────┘     └──────────────┘  ┌─▶│  Chat Server 2   │     │  (Queue)  │
                                       │  └──────────────────┘     └─────┬─────┘
                                       │           │                      │
                                       │     ┌─────▼──────┐        ┌─────▼──────┐
                                       │     │   Redis    │        │ Cassandra  │
                                       │     │ (conn map) │        │ (messages) │
                                       │     └────────────┘        └────────────┘
                                       │           │
                                       │     ┌─────▼──────┐        ┌────────────┐
                                       │     │  Presence  │        │  S3 + CDN  │
                                       └─────│  Service   │        │  (media)   │
                                             └────────────┘        └────────────┘
```

Don't just draw boxes. Name what each one does and why it exists. The interviewer wants to see your reasoning, not just your diagram.

---

## Step 3: The core problem — real-time message delivery

This is where most system design answers fall apart. And it's where mine did, until I understood it properly.

Here's the question that stumped me:

> *If User A is connected to Chat Server 1, and User B is connected to Chat Server 7 — how does the message get from A to B?*

The naive answer is "they go through the same server." But at WhatsApp scale — millions of concurrent connections — you can't route every user to a single server. You need dozens or hundreds of chat servers running in parallel. Users are scattered across them.

### The answer: connection mapping via Redis

Each chat server registers its connected users in a shared Redis store — a lookup table mapping `user_id → server_id`. When A sends a message:

```
User A
  │
  ▼
Chat Server 1  ──(1. lookup B's server)──▶  Redis
  │                                           │
  │◀──────(2. "B is on Server 7")─────────────┘
  │
  ├──(3. persist message)──▶  Kafka
  │                               │
  │                               ▼
  │                         Chat Server 7
  │                               │
  └───────────────────────────────▶  User B (delivered)
```

**Why WebSocket, not HTTP?**

HTTP is request-response. The client initiates, the server responds, the connection closes. For messaging, the server needs to push to the client at any time. WebSocket gives you a persistent, bidirectional connection — one TCP handshake, then it stays open. At 100M concurrent users, this is non-negotiable.

> **React Native note:** WebSocket connections on mobile need careful handling. Background transitions, network switches, and OS-level throttling all cause disconnects. Your RN client needs exponential backoff reconnection logic and a local queue to hold outgoing messages during disconnects — otherwise users lose messages when switching from WiFi to cellular.

---

## Step 4: Offline delivery — where good answers become great ones

Most candidates handle the happy path well. The senior signal is how you handle failure states. What happens when User B is offline?

```
User A  ──▶  Chat Server  ──▶  Kafka (message queued)
                                    │
                                    ├──▶  Cassandra (message persisted)
                                    │
                                    ▼
                              Push Service (APNs / FCM)
                                    │
                                    ▼
                              User B's device (notification)
                                    │
                              [B comes online]
                                    │
                                    ▼
                              Chat Server delivers queued messages
                                    │
                                    ▼
                              User B sends ACK ──▶  Server updates status
                                                         │
                                                         ▼
                                                    User A sees ✓✓
```

**The delivery receipt states:**

| State | Icon | What triggered it |
|-------|------|-------------------|
| Sent | ✓ | Server received the message from sender |
| Delivered | ✓✓ | Receiver's device acknowledged receipt |
| Read | 🔵🔵 | Receiver opened the conversation |

> "The best system design answers don't just describe the happy path. They anticipate failure and explain how the system recovers."

---

## Step 5: Storage design

At WhatsApp scale, storage decisions carry enormous consequences. This is where you demonstrate that you know *why* a tool is chosen, not just *that* it exists.

### Messages → Apache Cassandra

Messaging workloads are write-heavy (billions of messages per day) and time-ordered. Cassandra's wide-column model is built for this. Partition by `conversation_id`, cluster by a time-ordered UUID — giving you all messages for a conversation in sequential order, with writes distributed across the cluster.

### User data → PostgreSQL

Profiles, contacts, and settings are relational by nature and have much lower write volume. A traditional RDBMS handles this well and gives you proper foreign key constraints.

### Media → S3 + CDN

This is the one candidates most often get wrong. **Media should never flow through the chat server.**

Instead: the client uploads directly to S3, gets back a URL, and sends that URL as the message payload. The receiver fetches media from CloudFront or a similar CDN — distributed globally, close to the user.

```
┌─────────────────────────────────────────────┐
│        Chat Servers (routing layer)          │
└──────────────┬───────────┬──────────────────┘
               │           │           │
               ▼           ▼           ▼
        Cassandra     PostgreSQL    S3 + CDN
        (messages)    (users,       (media,
        partition:    contacts,     direct upload
        conv_id)      settings)     from client)
```

---

## Step 6: Group messaging — the fan-out problem

If a group has 256 members and someone sends a message, you need to deliver it to 255 people — potentially spread across 255 different chat servers. This is the *fan-out problem*.

**The wrong answer:** Client-side fan-out — having the sender's device send 255 individual messages. This puts unbounded load on the client connection, creates inconsistent delivery, and completely breaks for offline recipients.

**The right answer:** Server-side fan-out via Kafka.

```
User sends message
        │
        ▼
  Chat Server
  (looks up group membership)
        │
        ▼
      Kafka
  (one message, all recipient IDs)
        │
   ┌────┴────┬─────────┐
   ▼         ▼         ▼
Consumer  Consumer  Consumer
(routes   (routes   (routes
to Svr3)  to Svr7)  to Svr12)
   │         │         │
   ▼         ▼         ▼
 User C    User D    User E
```

This keeps the sender's connection clean, handles offline recipients through the existing queue mechanism, and lets you scale consumers independently of chat servers.

---

## Step 7: Presence service

"Last seen 3 minutes ago" sounds like a trivial feature. At 2 billion users, it's a distributed systems problem.

**How it works:**
- Clients send a heartbeat every ~30 seconds
- Presence Service stores `{user_id: last_seen_timestamp}` in Redis with a TTL
- If no heartbeat arrives within 60 seconds, the user is marked offline
- The Redis key expires naturally — no background cleanup job needed

Presence data is eventually consistent, and that's deliberate. WhatsApp adds a delay to "last seen" intentionally, for privacy reasons. Acknowledging this tradeoff in your interview — *"we're choosing availability over strict consistency here, and the UX actually benefits from the slight delay"* — is a strong signal.

---

## Step 8: Scaling to 100 million concurrent users

| Concern | Solution |
|---------|----------|
| Too many WebSocket connections per server | Horizontal scaling of chat servers; consistent hashing to assign users |
| Hot chat servers | Connection limits + auto-scaling with health checks |
| Redis becoming a bottleneck | Redis Cluster with key-based sharding |
| Kafka consumer lag | Partition by `conversation_id` for ordering; scale consumer groups independently |
| Media CDN costs | Tiered storage — hot media on CDN edge, cold on S3 Glacier |

---

## The meta-skill: how you talk through it matters as much as what you say

System design interviews are partially evaluated on your thought process, not just your architecture. A few habits that change how you're perceived:

**Name your tradeoffs explicitly.** Don't just say "I'd use Cassandra." Say: *"I'm choosing Cassandra over PostgreSQL here because of write throughput — we're doing billions of writes a day, and Cassandra's distributed write model handles that. The trade-off is that complex queries get harder, but we don't have those in the message flow."*

**Treat it as a conversation, not a presentation.** Every 5 minutes or so: *"Does this direction make sense, or would you like me to go deeper on any part?"* The best senior candidates treat the interviewer as a collaborator, not an audience.

**Draw before you explain.** Put rough boxes on the whiteboard first. Then narrate. It gives the interviewer something to anchor to, and stops you from getting lost in prose.

> **The moment I was unprepared for:** "How does the message get from Server 1 to Server 7?" — the Redis connection map is the answer. But the interviewer's real question was: *"Do you understand that in a distributed system, servers don't share memory?"* If you know that, the Redis answer flows naturally. If you don't, no amount of memorised architecture will save you.

---

## Wrapping up

Here's the full picture in one view:

```
Client (React Native)
    │
    ▼ WebSocket
Load Balancer
    │
    ▼
Chat Server ──── Redis (who's on which server)
    │
    ├──▶ Kafka ──▶ Consumer ──▶ Chat Server ──▶ Recipient
    │                │
    │                ├──▶ Cassandra (message store)
    │                └──▶ Push Service (offline users)
    │
    └──▶ Presence Service (heartbeats, last seen)

Media flow (separate):
Client ──▶ S3 (upload) ──▶ URL sent as message ──▶ CDN (recipient fetches)
```

This covers the core architecture — the 80% that gets you through a senior interview.

There's a 20% that separates a pass from a "strong hire": message ordering guarantees in distributed systems, how WhatsApp handles network partitions, and how to actually talk through all of this under real pressure.

**I've written that part in full on my blog → [subraatakumar.com/blog/whatsapp-system-design-complete](https://rnm.subraatakumar.com/blog/whatsapp-system-design-complete)**

---

*Found this useful? I write about React Native architecture, system design, and senior engineering at [subraatakumar.com](https://subraatakumar.com). I'm also building [RN Mastery](https://rnm.subraatakumar.com) — a contributor-driven learning platform for React Native developers.*
