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
│  ┌────────────┐  ┌─────────────┐  ┌──────────┐  ┌────────────┐  │
│  │ WebSocket  │  │  Local DB   │  │  Media   │  │   Push     │  │
│  │  Manager  │  │ (SQLite /   │  │  Upload  │  │  Handler   │  │
│  │           │  │  WatermelonDB│  │  (S3)   │  │ APNs/FCM   │  │
│  └─────┬─────┘  └─────────────┘  └────┬─────┘  └────────────┘  │
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
       └──────────┘      │ (APNs/FCM)  │
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
Message written to Local DB (SQLite / WatermelonDB / MMKV)
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

There's more to go deeper on — message ordering guarantees under network partitions, how to talk through all of this under real interview pressure, and the specific questions WhatsApp-style rounds use to probe your weak spots.

**I've written that part in full on my blog → [subraatakumar.com/blog/whatsapp-system-design-complete](https://rnm.subraatakumar.com/blog/whatsapp-system-design-complete)**

---

*Found this useful? I write about React Native architecture, system design, and senior engineering at [subraatakumar.com](https://subraatakumar.com). I'm also building [RN Mastery](https://rnm.subraatakumar.com) — a contributor-driven learning platform for React Native developers.*
