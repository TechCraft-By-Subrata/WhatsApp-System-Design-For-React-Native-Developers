# I Was Asked to Design WhatsApp in a Senior Interview. Here's the Full Breakdown — From the Mobile Architect's Lens.

*By Subrata Kumar — Cross-Platform Mobile Architect, React Native | 15 min read*

---

The interviewer said four words: **"Design a messaging app."** Then waited.

It was a senior mobile architect interview. I'd prepared. I'd read the usual system design resources. And then the interviewer said — casually, like it was nothing — "Walk me through how you'd design a messaging system. Like WhatsApp."

The problem wasn't that I didn't know anything. The problem was that I knew *about* WhatsApp. I knew what it did. What I wasn't prepared for was the moment the interviewer asked: *"Okay — if User A and User B are on different servers, how does the message get from one to the other?"*

Silence. Not the confident, thinking-out-loud kind. The kind where you feel the room change.

That question — that specific routing problem — is where most answers fall apart. But here's what I've learned since: as a mobile architect, you have an answer that backend engineers don't. You understand *both sides* of that connection. You know what happens on the server when a message is routed. And you know what happens on the device when the network drops, the app backgrounds, or the OS kills your socket.

Most system design articles on WhatsApp are written from a backend perspective. This one isn't. This is the full breakdown — backend *and* client — from the engineer who owns the layer that actually reaches the user.

---

## Step 1: Start with requirements, not architecture

The biggest mistake candidates make is drawing boxes before asking questions. The first 3–5 minutes of a system design interview should be you clarifying scope. Not drawing.

As a mobile architect, you have an additional lens here: **what does the client contract look like?** What does the app need from the backend to deliver a great experience?

**Functional Requirements (in scope):**
- 1:1 real-time messaging
- Group messaging (up to 256 members)
- Media sharing (images, video, documents)
- Online / offline status indicators
- Message delivery receipts (sent → delivered → read)
- Offline-first — messages typed while offline must send when connectivity restores

**Non-Functional Requirements:**
- 2 billion users, ~100M concurrent connections
- Message delivery under 100ms (perceived latency on device even lower via optimistic UI)
- Zero message loss — even when the receiver is offline, or the sender's app is killed mid-send
- End-to-end encrypted
- 99.99% availability
- Battery and data efficient on mobile — no aggressive polling

> **Interview tip:** Spending 3–5 minutes here signals engineering maturity. Notice the last two non-functional requirements — battery efficiency and data efficiency. A backend engineer might not think to mention these. A mobile architect should lead with them. They drive the choice of WebSocket over polling, and they show the interviewer you think about the full system.

---

## Step 2: High-level architecture

Once requirements are locked, give the 30,000-foot view before going deep. For a mobile architect, this means showing *both* the client layer and the backend — not just the server-side boxes.

```
┌──────────────────────────────────────────────────────────────────┐
│                        Mobile Client (RN)                        │
│  ┌────────────┐  ┌─────────────┐  ┌──────────┐  ┌────────────┐   │
│  │ WebSocket  │  │  Local DB   │  │  Media   │  │   Push     │   │
│  │  Manager   │  │ (SQLite /   │  │  Upload  │  │  Handler   │   │
│  │            │  │ WatermelonDB│  │  (S3)    │  │ APNs/FCM   │   │
│  └─────┬──────┘  └─────────────┘  └────┬─────┘  └────────────┘   │
└────────┼─────────────────────────────────────────────────────────┘
         │ WebSocket (persistent)          │ HTTPS (direct upload)
         ▼                                 ▼
┌──────────────┐                    ┌─────────────┐
│ Load Balancer│                    │   S3 + CDN  │
└──────┬───────┘                    │   (media)   │
       │                            └─────────────┘
       ▼
┌──────────────────────────────────────────────┐
│             Chat Servers (fleet)             │
└───┬──────────────┬──────────────┬────────────┘
    │              │              │
    ▼              ▼              ▼
┌───────┐    ┌──────────┐  ┌──────────────┐
│ Redis │    │  Kafka   │  │   Presence   │
│ conn  │    │ (queue)  │  │   Service    │
│ map   │    └────┬─────┘  └──────────────┘
└───────┘         │
             ┌────┴──────────────┐
             ▼                   ▼
       ┌──────────┐      ┌──────────────┐
       │Cassandra │      │ Notification │
       │(messages)│      │ Service      │
       └──────────┘      │ (APNs/FCM)   │
                         └──────────────┘
```

Name what each component does and why it exists — especially the client-side ones. For example: *"The WebSocket Manager on the client maintains a persistent connection and handles reconnection logic — because HTTP polling at this scale would drain battery and add unacceptable latency."*

---

## Step 3: The core problem — real-time message delivery

This is where most system design answers fall apart. And it's where mine did, until I understood it properly.

Here's the question that stumped me:

> *If User A is connected to Chat Server 1, and User B is connected to Chat Server 7 — how does the message get from A to B?*

The naive answer is "they go through the same server." But at WhatsApp scale — millions of concurrent connections — you can't route every user to a single server. You need dozens or hundreds of chat servers running in parallel. Users are scattered across them.

### The server-side answer: connection mapping via Redis

Each chat server registers its connected users in a shared Redis store — a lookup table mapping `user_id → server_id`. When A sends a message:

```
RN App (User A)
    │
    │ [1] sends message over WebSocket
    ▼
Chat Server 1
    │
    ├── [2] looks up "where is User B?" ──▶ Redis
    │                                          │
    │◀──────── "User B is on Server 7" ────────┘
    │
    ├── [3] persists message ──▶ Kafka
    │                               │
    │                               ▼
    │                         Chat Server 7
    │                               │
    │                               │ [4] pushes over WebSocket
    │                               ▼
    │                         RN App (User B) receives message
    │
    └── [5] Server 7 sends ACK back ──▶ User A sees ✓✓
```

### The client-side answer: what the RN app is doing

This is the part most candidates skip entirely. While the server routes the message, the client is doing its own work:

- **Optimistic UI** — the message appears in the conversation immediately on send, marked as pending (⏳). The user doesn't wait for a server round-trip to see their own message.
- **Local queue** — the message is written to local storage *before* it's sent over the socket. If the socket drops mid-send, the message isn't lost — it's retried when the connection restores.
- **ACK tracking** — the client maps each message to a local ID, then updates its status (pending → sent → delivered → read) as ACKs arrive from the server.

### Why WebSocket, not HTTP?

HTTP is request-response. The client initiates, the server responds, the connection closes. For messaging, the server needs to push to the client at any time. WebSocket gives you a persistent, bidirectional connection — one TCP handshake, then it stays open. At 100M concurrent users, polling is not an option — it would generate billions of unnecessary requests per minute and drain every battery in the fleet.

---

## Step 4: The mobile client layer — what the app is actually doing

This is the section that differentiates a mobile architect's answer from everyone else's. The backend handles routing. The client handles *reality* — dropped connections, backgrounded apps, OS restrictions, and users who type a message in a tunnel.

### 4a. WebSocket connection management

A WebSocket connection on mobile is fragile. The OS can kill it. WiFi to cellular handoffs drop it. Backgrounding suspends it. Your architecture needs to handle all of these gracefully.

```
App opens
    │
    ▼
Connect WebSocket
    │
    ├── Success ──▶ Register with server (user_id → server_id in Redis)
    │                   │
    │                   ▼
    │              Start heartbeat (every 30s) ──▶ keeps connection alive
    │                                               keeps presence updated
    │
    └── Failure ──▶ Exponential backoff retry
                    (1s → 2s → 4s → 8s... cap at 60s)
                    │
                    ▼
                   Retry ──▶ Success (resubscribe, sync missed messages)
```

**Cross-platform note:** iOS aggressively suspends background processes. When your app backgrounds, the WebSocket connection dies. This is why APNs push notifications exist — they're not just for UX, they're the *delivery mechanism* when the socket is unavailable. Android is more permissive but Doze mode still throttles background network activity. Your architecture must assume the socket is always *potentially* dead.

### 4b. Offline-first message handling

Users don't wait for connectivity before typing. Your app shouldn't either.

```
User types message (offline)
    │
    ▼
Message written to Local DB (SQLite / WatermelonDB / MMKV /Realm )
Status: PENDING
    │
    ▼ [when connection restores]
Message dequeued in order
    │
    ▼
Sent over WebSocket
    │
    ▼
Server ACK received ──▶ Local DB updated: PENDING → SENT ✓
```

The local database is your source of truth for message state. The server is the delivery mechanism. This is the mental model shift that separates a mobile architect from a backend engineer writing a mobile section.

**What to store locally:**
- All messages in a conversation (for instant load — no network round-trip to open a chat)
- Message status per message ID
- Pending outgoing queue (survives app kills)
- Last sync timestamp per conversation (for delta sync on reconnect)

### 4c. Media upload — direct to S3, never through the chat server

This is the one candidates most often get wrong. **Media should never flow through the chat server.**

```
User selects image
    │
    ▼
Client requests presigned S3 URL from server
    │
    ▼
Client uploads directly to S3 (HTTPS multipart)
    │
    ├── Show upload progress in UI (%)
    │
    ▼
S3 returns URL
    │
    ▼
Client sends message with URL as payload (not the image)
    │
    ▼
Recipient receives URL ──▶ fetches from CDN (geographically close)
                      ──▶ cached locally after first load
```

**Why this matters on mobile:** Multipart upload means large files can be resumed if the connection drops mid-upload. You're not re-uploading from scratch if the user goes through a tunnel at 40%.

### 4d. Message ordering on the client

Messages can arrive out of order — especially after a reconnect that fetches queued messages from Kafka alongside new live messages arriving over the socket. Rendering by arrival time is wrong. Rendering by server-assigned sequence number is right.

Each message carries a monotonically increasing sequence ID per conversation. The client sorts by this, not by local timestamp or arrival order. This is also what resolves the "message appeared above one I sent earlier" edge case.

### 4e. End-to-end encryption — the client owns the keys

The server never sees plaintext. Ever. Here's what the client is responsible for:

- **Key generation** — each device generates a public/private key pair on first launch. Private key stored in Keychain (iOS) or Keystore (Android). Never leaves the device.
- **Key exchange** — on first message to a contact, the Signal protocol performs a key exchange using the recipient's public key (fetched from the server's key registry).
- **Encryption before send** — every message is encrypted on-device before it hits the WebSocket. The server sees ciphertext and routes it. It cannot read it.
- **Decryption on receive** — the recipient's device decrypts using its private key. The server is a blind courier.

**Cross-platform note:** React Native doesn't have native crypto APIs by default. You'll use `react-native-quick-crypto` or a native module wrapping the platform's secure enclave. This is a non-trivial engineering problem — mention it, even if you don't go deep.

---

## Step 5: Offline delivery — where good answers become great ones

Most candidates handle the happy path well. The senior signal is how you handle failure states. What happens when User B is offline?

```
User A sends message
    │
    ▼
Chat Server checks Redis ──▶ User B is offline (no active connection)
    │
    ├──▶ Kafka (message queued, persisted in Cassandra)
    │
    ▼
Notification Service
    │
    ├──▶ APNs (iOS) ──▶ User B's device wakes
    └──▶ FCM  (Android) ──▶ User B's device wakes
                                    │
                              App comes to foreground
                                    │
                              WebSocket reconnects
                                    │
                              Fetches queued messages (in order, by sequence ID)
                                    │
                              Sends ACK to server
                                    │
                              Server marks delivered ✓✓ ──▶ User A notified
```

**The delivery receipt states:**

| State | Icon | Trigger |
|-------|------|---------|
| Pending | ⏳ | Written to local DB, not yet sent |
| Sent | ✓ | Server received and ACK'd |
| Delivered | ✓✓ | Receiver's device ACK'd receipt |
| Read | 🔵🔵 | Receiver opened the conversation |

The pending state is purely client-side — the server never knows about it. This is the optimistic UI layer that makes the app feel instant.

---

## Step 6: Storage — server side and client side

### Server-side storage

**Messages → Apache Cassandra**

Messaging workloads are write-heavy (billions of messages per day) and time-ordered. Cassandra's wide-column model handles this. Partition by `conversation_id`, cluster by sequence ID — all messages for a conversation in order, writes distributed across the cluster.

**User data → PostgreSQL**

Profiles, contacts, settings — relational, lower write volume. Standard RDBMS.

**Media → S3 + CDN**

Client uploads directly. Server stores the URL. Recipients fetch from CDN edge nodes close to them.

### Client-side storage

| What | Where | Why |
|------|-------|-----|
| Messages | SQLite / WatermelonDB | Relational queries, conversation history |
| Pending queue | MMKV / AsyncStorage | Fast writes, survives app kill |
| Media cache | File system + cache manager | Avoid re-downloading |
| Encryption keys | Keychain / Keystore | Secure enclave, never exported |
| Last sync timestamp | MMKV | Delta sync on reconnect |

---

## Step 7: Group messaging — the fan-out problem

If a group has 256 members and someone sends a message, you need to deliver it to 255 people — potentially spread across 255 different chat servers. This is the *fan-out problem*.

**The wrong answer — client-side fan-out:** The sender's device sends 255 individual messages. This puts unbounded load on the mobile connection (catastrophic on cellular), creates inconsistent delivery, and completely breaks for offline recipients.

**The right answer — server-side fan-out via Kafka:**

```
RN App sends one message
    │
    ▼
Chat Server looks up group membership
    │
    ▼
Publishes to Kafka (one event, all 255 recipient IDs)
    │
    ├──▶ Consumer ──▶ routes to Chat Server 3  ──▶ User C
    ├──▶ Consumer ──▶ routes to Chat Server 7  ──▶ User D (or queues if offline)
    └──▶ Consumer ──▶ routes to Chat Server 12 ──▶ User E
```

The client sends one message. The server handles the multiplication. This is the correct mobile-server contract.

---

## Step 8: Presence service

"Last seen 3 minutes ago" sounds like a trivial feature. It's actually a distributed heartbeat system touching every connected device simultaneously.

**Server side:**
- Presence Service stores `{user_id: last_seen_timestamp}` in Redis with a TTL
- No heartbeat for 60 seconds → user marked offline, Redis key expires

**Client side:**
- App sends a heartbeat every 30 seconds over the existing WebSocket (no extra connection needed)
- On app background (iOS) → heartbeat stops → user goes offline after TTL
- On app foreground → WebSocket reconnects → heartbeat resumes → user back online

**The tradeoff to mention:** Presence is eventually consistent by design. WhatsApp deliberately delays "last seen" updates — it's a privacy feature, not a bug. Choosing availability over strict consistency here is the right call, and naming it explicitly shows architectural maturity.

---

## Step 9: Scaling to 100 million concurrent users

| Concern | Solution |
|---------|----------|
| Too many WebSocket connections per server | Horizontal scaling of chat servers; consistent hashing to assign users |
| Chat server goes down | Client detects disconnect → reconnects to new server → Redis re-registers connection |
| Redis bottleneck | Redis Cluster with key-based sharding |
| Kafka consumer lag | Partition by `conversation_id`; scale consumer groups independently |
| CDN media costs | Tiered storage — hot media on edge, cold on S3 Glacier |
| Client reconnect storm (all users reconnect at once after an outage) | Jittered exponential backoff on client — staggers reconnects across time |

That last row is a mobile-specific scaling concern. A backend engineer won't mention it. You should.

---

## The meta-skill: how you talk through it matters as much as what you say

System design interviews are evaluated on thought process, not just architecture. A few habits that change how you're perceived:

**Lead with mobile, then go backend.** Most candidates describe the server first and add mobile as an afterthought. Flip it. Start with what the app needs to deliver a great experience, then design the backend to serve that contract. It signals ownership of your domain.

**Name your tradeoffs explicitly.** Don't say "I'd use Cassandra." Say: *"I'm choosing Cassandra over PostgreSQL here because of write throughput — billions of messages a day, and Cassandra's distributed write model handles that. The trade-off is that complex queries get harder, but we don't have those in the message flow."*

**Treat it as a conversation, not a presentation.** Every 5 minutes or so: *"Does this direction make sense, or would you like me to go deeper on any part?"* The best senior candidates treat the interviewer as a collaborator.

**Draw before you explain.** Client layer first, then backend. It gives the interviewer a map and grounds everything that follows.

> **The moment I was unprepared for:** "How does the message get from Server 1 to Server 7?" — the Redis connection map is the answer. But the interviewer's real question was: *"Do you understand that in a distributed system, servers don't share memory?"* If you know that, the Redis answer flows naturally. If you don't, no amount of memorised architecture will save you.

---

## Wrapping up — the full picture

```
┌──────────────────────────────────────────────────────┐
│                  Mobile Client (RN)                  │
│                                                      │
│  WebSocket Manager  ←→  Local DB (messages, queue)  │
│  Media Uploader ──────────────────────▶ S3           │
│  Push Handler (APNs/FCM)                             │
│  Crypto Layer (Keychain/Keystore)                    │
└────────────────────┬─────────────────────────────────┘
                     │ WebSocket
                     ▼
              Load Balancer
                     │
                     ▼
             Chat Server Fleet
            /        │        \
           ▼         ▼         ▼
        Redis      Kafka    Presence
      (conn map)  (queue)   Service
                    │
              ┌─────┴──────┐
              ▼             ▼
          Cassandra    Notification
          (messages)   (APNs / FCM)

S3 + CDN ◀── direct upload from client
          ──▶ direct fetch by recipient
```

The backend routes messages. The client delivers the experience. A mobile architect owns both halves of that sentence — and that's what makes the answer complete.

---

# What Most Mobile Architects Miss in the WhatsApp System Design Interview

---

## The gap most candidates leave open

Here's what I've observed across architect-level interviews: candidates who've done their homework can explain the server-side architecture reasonably well. They know about WebSockets, they've heard of Kafka, they can sketch a rough diagram.

Where they fall apart is in three specific places:

1. **Message ordering** — what guarantees that messages arrive in the right sequence, especially after a reconnect?
2. **Network partitions** — what happens when parts of your distributed system can't talk to each other? What do you sacrifice?
3. **The client under pressure** — what is the RN app *actually* doing during all of this, and why does it matter in an architect interview?

Let's go through each one.

---

## Deep Dive 1: Message ordering — harder than it looks

You've designed the routing. Messages flow from User A's chat server through Redis to User B's chat server and arrive over WebSocket. Simple enough.

Now here's what the interviewer asks next: *"How do you guarantee that messages arrive in the correct order?"*

Most candidates say: "Use timestamps."

That's the wrong answer.

### Why timestamps fail in distributed systems

Imagine User A sends two messages in quick succession — "Hey" and "Are you there?" — from their device. Both messages hit Chat Server 1. Chat Server 1 publishes both to Kafka. Two different Kafka consumers pick them up and route them to Chat Server 7. Chat Server 7 pushes them to User B's WebSocket.

The problem: those two consumers run independently. Consumer 2 might finish faster than Consumer 1. "Are you there?" arrives before "Hey." The conversation is now out of order.

"But wait," you say, "just use the send timestamp from the client."

New problem: clocks on distributed servers drift. NTP helps, but it doesn't eliminate drift entirely. Two messages sent milliseconds apart can have timestamps that swap depending on which server processed them. At WhatsApp's scale — billions of messages a day — this isn't a theoretical edge case. It's a constant reality.

### The right answer: monotonic sequence IDs per conversation

Each conversation has a server-assigned, monotonically increasing sequence counter stored in Redis. Every message that arrives for a conversation gets the next sequence number *before* it's published to Kafka.

```
Message arrives at Chat Server
    │
    ▼
Atomic increment on Redis key: conv_{conversation_id}_seq
    │
    ▼ returns seq = 1042
    │
Message tagged with seq: 1042
    │
    ▼
Published to Kafka (with seq in payload)
    │
    ▼
Delivered to recipient with seq: 1042
```

The atomic Redis increment guarantees that no two messages in the same conversation ever get the same sequence number, and that the numbers are strictly increasing regardless of which chat server handled the message.

### What the client does with this

This is the mobile architect angle. The server assigns sequence numbers. The client is responsible for *using* them correctly.

When messages arrive on the WebSocket, the RN app doesn't render them in arrival order. It inserts them into a sorted local buffer by sequence number, then renders from that buffer.

```
Messages arrive from WebSocket (potentially out of order):
seq: 1044, seq: 1042, seq: 1043

Local buffer sorts by seq:
[1042] "Hey"
[1043] "Are you there?"
[1044] "Just checking in"

FlatList renders from sorted buffer ✓
```

This also handles the reconnect scenario cleanly. When User B's app reconnects after being offline, it fetches missed messages from the server starting from its last known sequence number — not from a timestamp. The delta sync is precise and ordered.

> **Interview tip:** When you explain message ordering, mention *both* the server mechanism (Redis atomic counter) and the client mechanism (sorted buffer by sequence ID). Most candidates only talk about one side. Mentioning both shows you genuinely think in full-stack system terms — which is exactly what a mobile architect interview is testing.

---

## Deep Dive 2: Network partitions — what do you sacrifice?

This is where system design interviews get philosophical. And it's where a lot of candidates either dodge the question or give a textbook answer that doesn't land.

The interviewer will ask something like: *"What happens when your chat servers can't reach Kafka? Or when Redis goes down? How does your system behave?"*

### The CAP theorem — not as a definition, but as a decision

You've probably heard of the CAP theorem: in a distributed system, you can only guarantee two of three properties — Consistency, Availability, and Partition tolerance. Since network partitions are a reality you can't avoid, the real choice is between Consistency and Availability.

WhatsApp chooses **Availability**. Here's what that looks like in practice.

**Scenario: Chat Server can't reach Kafka**

```
User A sends message
    │
    ▼
Chat Server 1 tries to publish to Kafka
    │
    ✗ Kafka unreachable (network partition)
    │
    ▼
Chat Server stores message locally (in-memory buffer)
    │
    ▼
Returns optimistic ACK to User A
(User A sees ✓ — "sent")
    │
    ▼ [when Kafka recovers]
Chat Server flushes buffer to Kafka
    │
    ▼
Message delivered to User B
User A eventually sees ✓✓
```

The user experience is preserved. The message appears sent. The system recovers when the partition heals.

The risk: in an extreme failure — chat server crashes before Kafka recovers — that message is lost. WhatsApp accepts this tradeoff. The alternative (refusing to show "sent" until Kafka confirms) would make the app feel broken every time there's a network hiccup.

**Scenario: Redis goes down**

Redis holds the connection map (`user_id → server_id`). If Redis is unavailable, chat servers can't route messages to each other.

The response: fall back to broadcasting. The sending chat server publishes the message to Kafka. All chat servers consume from Kafka and check if the recipient is connected to *them*. Inefficient at scale, but it keeps messages flowing during a Redis outage.

This is a temporary degraded mode — not a permanent architecture — and naming it as such in an interview shows maturity.

### The mobile client during a partition

Here's the client-side angle that no one covers.

When the server returns an optimistic ACK during a partition, the client shows ✓. But the client doesn't *know* it's an optimistic ACK during a partition versus a real ACK after successful delivery. It just sees ✓.

This is intentional. The client's job is to maintain a consistent, calm UI. The server's job is to eventually deliver. The contract between them is: *if the server ACKs, the client trusts it.*

What the client *does* do is maintain a timeout on the ✓ → ✓✓ transition. If "delivered" doesn't arrive within a reasonable window, the client doesn't panic — it just leaves the message at ✓ until the ACK arrives. No error state, no retry prompt. Just patience, because the system is designed to eventually deliver.

> **Interview tip:** When discussing CAP, don't just recite the theorem — explain the *user experience consequence* of each choice. "If we choose consistency, the user sees a spinner or an error during a partition. If we choose availability, the user sees ✓ and the message catches up later. For a messaging app, availability is the right call — a spinner feels broken, a slight delay in ✓✓ is invisible." That reasoning is what interviewers remember.

---

## Deep Dive 3: The client architecture under real pressure

This is the section that exists nowhere else. Not in backend system design courses. Not in interview prep books. Because it's the layer that only a mobile architect lives in.

Let me walk through what the RN client is *actually* doing during a busy messaging session — not as a diagram, but as a story.

### The life of a message on the client

You open WhatsApp. You type "Hey, are you free tonight?" and hit send.

Before that message travels a single byte over the network, four things happen on your device:

**1. Local write first**
The message is written to the local database immediately — SQLite or WatermelonDB — with a generated local ID, the text, and a status of `PENDING`. This is synchronous and happens before the WebSocket send. If your app is killed at this exact moment, the message survives.

**2. Optimistic render**
The message appears in your conversation list instantly. No spinner. No waiting. You see your own message in the UI before the server knows it exists. This is the optimistic UI layer — the thing that makes WhatsApp feel faster than it is.

**3. Queue for send**
The message is added to an outgoing queue. The queue manager picks it up, checks if the WebSocket is connected, and sends it. If the WebSocket is down, the message stays in the queue. When the connection restores, the queue drains in order.

**4. Status lifecycle**
As ACKs arrive from the server, the local database is updated:

```
PENDING (local only)
    ↓ WebSocket send succeeds
SENT ✓ (server received)
    ↓ Recipient device ACKs
DELIVERED ✓✓ (recipient's device has it)
    ↓ Recipient opens conversation
READ 🔵🔵 (recipient saw it)
```

Each state transition is a server event that the client handles by updating the local DB and re-rendering only the affected message row — not the entire list.

### The reconnect story

Now imagine you send that message and immediately walk into a tunnel. The WebSocket drops.

On the client:
- The message is in the outgoing queue with status `PENDING`
- `NetInfo` fires a connectivity change event
- The WebSocket Manager starts its reconnect cycle: wait 1s, try, fail, wait 2s, try, fail, wait 4s...
- The UI shows nothing alarming. The message sits there with ⏳. No error. No red text.

You come out of the tunnel. Connectivity restores.

- WebSocket reconnects successfully
- Client re-registers with the server (new `user_id → server_id` entry in Redis)
- Client sends its last known sequence number: "give me everything from seq 1041 onwards"
- Server flushes queued incoming messages in order
- Outgoing queue drains — your "Hey, are you free tonight?" finally sends
- Status updates: PENDING → SENT ✓ → DELIVERED ✓✓

All of this happens in the background. You never saw an error. You never had to resend.

**That** is what offline-first architecture feels like from the user's seat. And it's what you're designing when you spec the client layer in an architect interview.

### The battery and data efficiency story

Two things that backend system design articles never mention — because they don't affect the backend:

**Battery:** A persistent WebSocket connection consumes power. The heartbeat (sent every 30 seconds) keeps the connection alive but also keeps the radio awake. WhatsApp manages this by coalescing heartbeats with other network activity when possible, and by accepting that the connection will die in the background on iOS — relying on APNs as the wake-up mechanism instead of fighting the OS.

**Data:** Media never flows through the WebSocket. Images, videos, and documents are uploaded directly to S3 and downloaded from CDN. The WebSocket carries only message payloads — text and metadata. For a user on a limited data plan, this matters enormously.

> **Interview tip:** Mentioning battery and data efficiency unprompted is one of the clearest signals that you're thinking as a mobile architect, not a backend engineer who learned some mobile. These concerns don't exist in backend system design. They exist in the layer you own.

### The encryption edge case nobody mentions — uninstall and reinstall

The Medium article covered the basics of E2E encryption: the client generates a public/private key pair, the private key never leaves the device, the server is a blind courier that only ever sees ciphertext.

But here's the question interviewers use to probe whether you've *actually* thought this through:

*"What happens to the encryption when a user uninstalls and reinstalls the app?"*

Here's what happens:

```
User uninstalls WhatsApp
    │
    ▼
App data wiped
Private key deleted from Keychain / Keystore ← gone forever
    │
    ▼
Server still holds the OLD public key
All contacts still have the OLD public key cached locally
    │
    ▼
User reinstalls
    │
    ▼
Brand new key pair generated on device
New public key registered with server (replaces old one)
    │
    ▼
Server notifies all contacts:
"This user's security code has changed"
    │
    ▼
Contacts' apps re-fetch the new public key
New encrypted sessions established
```

That notification you've probably seen in WhatsApp — *"Subrata's security code changed. Tap to learn more."* — isn't a bug. It's the system doing exactly what it should: telling every contact that the key they were using to encrypt messages is no longer valid, and they need to re-establish a secure session with the new key.

**The trust problem this creates**

This is where it gets architecturally interesting. If the server can swap a public key — even legitimately, as in the reinstall case — a *compromised* server could theoretically swap your contact's public key with its own. It would then decrypt your incoming messages, read them, re-encrypt with the real recipient's key, and forward them. A perfect man-in-the-middle attack.

WhatsApp's defence is the **safety number** (also called the security code) — a fingerprint derived from both parties' current public keys. If you and your contact compare safety numbers in person and they match, you've cryptographically verified that no key swap occurred between you and the server.

Most users never do this. WhatsApp knows this. It's a known limitation they accept — the UX cost of forcing key verification would outweigh the security benefit for the vast majority of users who aren't high-value targets.

```
Safety number verification flow:
User A and User B meet in person
    │
    ▼
Both open WhatsApp → Contact → Encryption → Safety Number
    │
    ▼
Compare the 60-digit number (or scan each other's QR code)
    │
    ├── Numbers match ──▶ ✅ No MITM, keys are authentic
    └── Numbers differ ──▶ ⚠️ Key mismatch — potential compromise or recent reinstall
```

> **Interview tip:** Raising the reinstall edge case unprompted — and then connecting it to the MITM attack surface and safety number defence — is the kind of answer that makes an interviewer put down their pen and lean forward. It shows you're not reciting an architecture. You're reasoning about it. That's the mobile architect signal.

---

## Putting it all together

Here's what the full client architecture looks like when you add these deep dives to the picture:

```
User Action (type + send)
    │
    ▼
Local DB write (PENDING) ──▶ Optimistic UI render
    │
    ▼
Outgoing Queue
    │
    ├── WebSocket connected ──▶ Send immediately
    │                               │
    │                               ▼
    │                         Server ACK ──▶ DB update: PENDING → SENT ✓
    │                               │
    │                               ▼
    │                    Recipient ACK ──▶ DB update: SENT → DELIVERED ✓✓
    │                               │
    │                               ▼
    │                    Read receipt ──▶ DB update: DELIVERED → READ 🔵🔵
    │
    └── WebSocket disconnected
            │
            ▼
        Message stays PENDING in queue
            │
        [NetInfo: connection restored]
            │
            ▼
        WebSocket reconnects ──▶ Re-register with server
            │
            ▼
        Delta sync (last seq → now) ──▶ Incoming messages ordered by seq
            │
            ▼
        Queue drains ──▶ Message sends ──▶ Status lifecycle begins
```

This is the diagram that tells an interviewer you don't just know distributed systems. You know what distributed systems look like from inside a mobile app — which is exactly what they're hiring a mobile architect to understand.

---

## What this means in the interview room

When you walk into a mobile architect system design interview and the question is "design WhatsApp," most candidates will give the backend answer with mobile footnotes.

You give the mobile answer with backend context.

That means:
- **Lead with the client contract** — what does the app need from the backend to deliver a great user experience?
- **Design the backend to serve that contract** — not the other way around
- **Name mobile-specific concerns explicitly** — battery, data, OS restrictions, reconnect storms, offline-first
- **Show both sides of every decision** — what the server does, and what the client does in response

The interviewer isn't just evaluating whether you know Kafka. They're evaluating whether you think in systems. And a mobile architect who thinks in full-stack systems — client *and* server, UX *and* infrastructure — is genuinely rare.

That's the position you're in. Own it.

---

## What's next

I'm building a step-by-step System Design course for React Native developers on RN Mastery — not a generic course, but one built around the real questions asked in mobile and full-stack architect interviews.

Every question walked through end-to-end. Server side *and* client side. With diagrams, decision frameworks, and mock interview scripts that tell you what to say, what to avoid, and how to handle the moments you don't know the answer.

The course isn't live yet. But if this article gave you something — join the waitlist. You'll be the first to know when it launches, and waitlist members get early access and a discount.

**[Join the waitlist → rnm.subraatakumar.com/courses/system-design]([https://rnm.subraatakumar.com/system-design](https://rnm.subraatakumar.com/courses/system-design))**

---

*Subrata Kumar is a Cross-Platform Mobile Architect with 10+ years in software engineering and 6+ years building React Native production apps across healthcare, e-commerce, and EdTech. He is the creator of [RN Mastery](https://rnm.subraatakumar.com) — a learning platform for React Native developers who want to go from developer to architect.*
