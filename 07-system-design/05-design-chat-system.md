# System Design: Chat System

> **18 questions**

- Requirements and estimation: daily message volume, peak concurrent WebSocket connections, storage growth per year, read-to-write ratio, bandwidth per connection
- API design: REST endpoints and WebSocket event schemas
- Data models: users, conversations (1:1 and group), messages, read receipts
- High-level architecture: API gateway, chat service, presence service, notification service, message store
- WebSocket vs long polling vs SSE vs HTTP polling for real-time delivery
- Message flow: end-to-end 1:1 delivery, offline user handling
- Group messaging: fan-out on write vs read, small vs large group strategies, roles, permissions, membership changes
- Message storage: NoSQL (Cassandra, DynamoDB) partition/sort key design, pagination
- Message routing: centralized connection registry (Redis), pub/sub, consistent hashing
- Message ordering: per-conversation ordering, sequence numbers, server-assigned timestamps
- Delivery guarantees: sent/delivered/read, at-least-once, idempotency keys, deduplication
- WebSocket gateway: connection lifecycle, JWT auth handshake, heartbeats, reconnection
- Read receipts, typing indicators, and unread counts: event flow, per-user read positions, throttling for large groups
- Scaling WebSocket servers: graceful draining, sticky sessions vs connection registry
- Failure scenarios: gateway failure (reconnection + catch-up), message store unavailability (write-ahead queue, retry), presence service crash (graceful degradation, stale state TTL)

---

## Requirements & Estimation

<details>
<summary>1. You're asked to design a chat system like WhatsApp or Slack — how do you gather functional and non-functional requirements, and how do you estimate message volume, concurrent WebSocket connections, storage growth per year, and bandwidth needs to establish the scale constraints that will drive every architectural decision?</summary>

**Functional requirements** — clarify with the interviewer:

- 1:1 messaging and group chats (max group size?)
- Real-time delivery with online/offline support
- Message history and persistence
- Read receipts, typing indicators, presence (online/offline/last seen)
- Push notifications for offline users
- Media/file sharing (or text-only to start?)

**Non-functional requirements:**

- Low latency: messages delivered in under 200ms for online users
- High availability: messaging should never go down entirely
- Ordering: messages appear in correct order within a conversation
- Durability: no message loss once accepted by the server
- At-least-once delivery with client-side deduplication

**Back-of-envelope estimation** (assuming WhatsApp-scale: 500M DAU):

| Metric | Calculation | Result |
|---|---|---|
| Daily messages | 500M users x 40 msgs/day | 20B messages/day |
| Messages per second | 20B / 86,400 | ~230K msg/s (peak ~3x = 700K) |
| Concurrent WebSocket connections | 500M x 10% online at any time | 50M concurrent connections |
| Storage per year | 20B msgs x 100 bytes avg x 365 | ~730 TB/year (text only) |
| Bandwidth per connection | 1 msg/min x 200 bytes (message payload + WebSocket frame header + protocol metadata) | ~3.3 bytes/s per conn (~160 KB/day) |
| Total ingress bandwidth | 700K msg/s x 200 bytes | ~140 MB/s peak |

**Key scale constraints these numbers drive:**

- 50M concurrent WebSocket connections means a large fleet of gateway servers (each handles ~50-100K connections), so you need a connection registry
- 700K msg/s peak means the message store must handle massive write throughput — rules out a single relational database, points toward Cassandra/DynamoDB
- 730 TB/year means you need a storage strategy with compaction, TTL policies, or tiered storage
- Read-to-write ratio: typically 1:1 to 5:1 (users read messages roughly as often as they're sent, but group messages are read by many), which affects fan-out strategy

The estimation step matters because it tells you *which* problems are real. At 10K users, a single PostgreSQL instance handles everything. At 500M DAU, almost every component needs horizontal scaling and a distributed approach.

</details>

<details>
<summary>2. Design the API surface for a chat system — what REST endpoints do you need for non-real-time operations (user management, conversation CRUD, message history), what WebSocket event schemas handle real-time communication (send message, receive message, typing, presence updates), and why do you split the API this way instead of putting everything over WebSocket or everything over REST?</summary>

**REST endpoints** (request/response, non-real-time):

```
POST   /api/users                          # register
POST   /api/auth/login                     # get JWT
GET    /api/conversations                  # list user's conversations
POST   /api/conversations                  # create conversation (1:1 or group)
GET    /api/conversations/:id/messages?cursor=X&limit=50  # paginated history
POST   /api/conversations/:id/members      # add member to group
DELETE /api/conversations/:id/members/:uid  # remove member
PUT    /api/conversations/:id              # update group name, avatar
POST   /api/media/upload                   # upload file/image, returns URL
```

**WebSocket event schemas** (real-time, bidirectional):

```typescript
// Client → Server
{ type: "message.send", conversationId: string, clientMsgId: string, content: string, contentType: "text" | "image" }
{ type: "typing.start", conversationId: string }
{ type: "typing.stop", conversationId: string }
{ type: "message.ack", messageId: string }           // client confirms receipt
{ type: "message.read", conversationId: string, messageId: string }  // read receipt
{ type: "ping" }

// Server → Client
{ type: "message.new", messageId: string, conversationId: string, senderId: string, content: string, timestamp: number, clientMsgId: string }
{ type: "message.sent", clientMsgId: string, messageId: string, timestamp: number }  // server ACK
{ type: "typing", conversationId: string, userId: string, isTyping: boolean }
{ type: "presence", userId: string, status: "online" | "offline", lastSeen?: number }
{ type: "message.delivered", messageId: string, userId: string }
{ type: "message.read", conversationId: string, userId: string, readUpTo: string }
{ type: "pong" }
```

**Why the split:**

- **REST for CRUD/history**: These are request-response operations that benefit from standard HTTP semantics — caching (ETags on history), retries, content negotiation, load balancer routing, and standard auth middleware. They're also stateless, so any server can handle them.
- **WebSocket for real-time**: Typing indicators, presence, and message delivery need sub-100ms push latency. HTTP polling would waste bandwidth and add latency. SSE is server-to-client only, so you'd still need a second channel for sending.
- **Why not all-WebSocket**: You *could* route everything through WebSocket, but you lose HTTP caching, standard middleware stacks, CDN integration for media, and the simplicity of stateless request routing for operations like fetching history. WebSocket connections are stateful and harder to load-balance.
- **Why not all-REST**: Polling for new messages is wasteful. Even at 1-second intervals, 50M users polling means 50M requests/second for what's usually an empty response. WebSocket inverts the model — the server pushes only when there's something to deliver.

</details>

<details>
<summary>3. Design the core data models for users, conversations (supporting both 1:1 and group chats), messages, and read receipts — what fields does each entity need, how do you model the relationship between conversations and participants, and why does the choice of 1:1 vs group conversation model affect schema design decisions downstream (e.g., should 1:1 chats be a special case of group chats or a separate model)?</summary>

**Recommended approach: unify 1:1 and group as a single conversation model.** This simplifies the codebase significantly — every message goes to a conversation, every conversation has members. A 1:1 chat is just a conversation with exactly 2 members and `type: "direct"`.

```typescript
// User
{
  userId: string,           // UUID
  username: string,
  displayName: string,
  avatarUrl: string | null,
  lastSeen: number,         // epoch ms
  status: "online" | "offline",
  createdAt: number
}

// Conversation
{
  conversationId: string,   // UUID
  type: "direct" | "group",
  name: string | null,      // null for direct, group name for groups
  createdBy: string,        // userId
  createdAt: number,
  updatedAt: number         // last activity timestamp, used for sorting conversation list
}

// ConversationMember (many-to-many)
{
  conversationId: string,
  userId: string,
  role: "owner" | "admin" | "member",
  joinedAt: number,
  lastReadMessageId: string | null,  // tracks read position
  lastReadAt: number | null,
  muted: boolean,
  notifications: "all" | "mentions" | "none"
}

// Message
{
  messageId: string,         // UUID or ULID/Snowflake (sortable)
  conversationId: string,    // partition key for storage
  senderId: string,
  content: string,
  contentType: "text" | "image" | "file" | "system",
  clientMsgId: string,       // client-generated, used for deduplication and ACK matching
  createdAt: number,         // server-assigned timestamp
  sequenceNum: number,       // per-conversation ordering
  metadata: Record<string, any> | null  // reactions, edit history, link previews
}
```

**Why unified model over separate 1:1 and group models:**

- One message table, one delivery pipeline, one history API. No branching logic everywhere.
- Upgrading a 1:1 chat to a group (adding a third person) is trivial — add a member, change type.
- The cost is minimal: direct chats carry a `name: null` field and a membership table with exactly 2 rows.

**Why it affects downstream design:**

- **Storage**: If 1:1 were separate, you might partition by `(user1, user2)` tuple. Unified model always partitions by `conversationId`, which is simpler.
- **Fan-out**: Group fan-out logic applies to all conversations — for direct chats it just means fan-out to 2 members.
- **Permissions**: Group permission checks (can this user post?) still run for direct chats but trivially pass.
- **Finding existing direct chats**: You need a lookup index `(userId_A, userId_B) → conversationId` to prevent creating duplicate 1:1 conversations. Sort the user IDs to make this deterministic.

**Read position tracking**: Note that read receipts are stored on the `ConversationMember` row as `lastReadMessageId` rather than as a separate row per message. This is covered in detail in question 15.

</details>

## High-Level Architecture

<details>
<summary>4. Sketch the high-level architecture of a chat system showing the API gateway, chat service, presence service, notification service, and message store — why does each component exist as a separate service instead of being combined, what are the communication patterns between them, and what would you lose by merging any two of these components?</summary>

**Architecture diagram:**

```
Clients (mobile/web)
       │
       ├── HTTPS ──► [API Gateway / Load Balancer]
       │                     │
       │              ┌──────┼──────────┐
       │              ▼      ▼          ▼
       │         [REST API] [Auth]  [Media Service]
       │           │                    │
       │           ▼                    ▼
       │      [Chat Service]        [Object Storage / CDN]
       │           │
       └── WSS ──► [WebSocket Gateway Fleet]
                        │
                   [Connection Registry (Redis)]
                        │
                   ┌────┼────────┬──────────────┐
                   ▼    ▼        ▼              ▼
            [Chat Service] [Presence Service] [Notification Service]
                   │              │                    │
                   ▼              ▼                    ▼
            [Message Store]  [Redis/In-memory]   [Push Gateway (APNs/FCM)]
            (Cassandra/DDB)
                   │
            [Message Queue (Kafka)]  ◄── connects Chat, Notification, Presence
```

**Why each component is separate:**

| Component | Responsibility | Why separate |
|---|---|---|
| **WebSocket Gateway** | Manages persistent connections, protocol handling | Stateful (holds connections), scales differently than stateless services. Needs its own deployment/draining strategy. |
| **Chat Service** | Message processing, validation, storage, delivery routing | Core business logic. Changes frequently. Needs to scale based on message throughput, not connection count. |
| **Presence Service** | Online/offline status, last seen | High-frequency ephemeral state (heartbeats every 30s from all connected users). Would overwhelm the message store if coupled with it. |
| **Notification Service** | Push notifications to offline users | Depends on external services (APNs, FCM) with their own rate limits, failures, and retry logic. Different SLA from real-time delivery. |
| **Message Store** | Persistent message storage | Optimized for write-heavy, append-only workloads. Storage layer decisions (Cassandra vs DynamoDB) shouldn't be coupled to application logic. |

**Communication patterns:**

- Client → WebSocket Gateway: persistent WebSocket connection
- WebSocket Gateway → Chat Service: internal RPC (gRPC or HTTP) for message processing
- Chat Service → Message Store: direct database writes
- Chat Service → Kafka: publishes message events
- Kafka → Notification Service: consumes events for offline users, sends push
- Kafka → Presence Service: consumes connect/disconnect events
- Chat Service → Connection Registry (Redis): looks up which gateway server holds the recipient's connection
- Chat Service → WebSocket Gateway: pushes message to the correct gateway for delivery

**What you'd lose by merging:**

- **Gateway + Chat Service**: You can't scale connections independently of message processing. Deploying a chat service fix would drain all WebSocket connections.
- **Chat Service + Presence Service**: Presence heartbeats (50M users x every 30s = 1.6M ops/s) would compete with message processing resources. Presence is best-effort; messaging is durable — different reliability requirements.
- **Chat Service + Notification Service**: A push notification provider outage (APNs down) would back-pressure into your real-time message pipeline. Separating them with a queue provides isolation.

</details>

<details>
<summary>5. Compare WebSocket, long polling, Server-Sent Events (SSE), and HTTP polling for real-time message delivery in a chat system — what are the tradeoffs of each in terms of latency, server resource usage, connection overhead, and bidirectional communication, why do most production chat systems choose WebSocket, and when might you still need a fallback transport?</summary>

| Aspect | HTTP Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| **Latency** | Up to polling interval (1-5s) | Near real-time (new request after each response) | Real-time server→client | Real-time bidirectional |
| **Connection overhead** | New TCP+TLS per request | One open request at a time, re-established after each response | Single persistent HTTP/2 connection (server→client) | Single persistent TCP connection |
| **Bidirectional** | No (client→server via separate POST) | No (client→server via separate POST) | No (server→client only) | Yes, native |
| **Server resources** | Low per-request, high aggregate (many empty responses) | Moderate (holds open connections, but releases on each response) | Low (one connection, server push only) | Low per-connection, but stateful (memory for each conn) |
| **Protocol** | Standard HTTP | Standard HTTP | HTTP with `text/event-stream` | Upgrade from HTTP, then raw frames |
| **Proxy/firewall compat** | Excellent | Good | Good (uses HTTP) | Sometimes blocked by corporate proxies |
| **Browser support** | Universal | Universal | No IE support | Universal modern |

**Why WebSocket wins for chat:**

1. **Bidirectional**: Chat inherently needs both directions — sending and receiving messages, typing indicators, delivery ACKs. WebSocket handles both on one connection.
2. **Low latency**: No request/response overhead per message. A frame is sent immediately.
3. **Efficient**: One persistent connection vs establishing a new HTTP request for every interaction. At 50M concurrent users, the overhead difference is massive.
4. **Low overhead per frame**: WebSocket frames have 2-14 bytes of overhead vs HTTP headers (hundreds of bytes per request).

**When you need a fallback:**

- Corporate networks with proxies that terminate or block WebSocket upgrades
- Degraded network conditions where a simpler protocol recovers faster

Typical fallback chain: WebSocket → SSE + REST POST → long polling. Libraries like Socket.IO implement this automatically, though at scale most teams use raw WebSocket with a thin fallback layer rather than Socket.IO's overhead.

</details>

<details>
<summary>6. Walk through the end-to-end flow of a 1:1 message from the moment User A hits send to the moment User B sees it on screen — cover every service the message passes through, where it gets persisted, how the recipient's connection is located, and what happens differently when User B is offline (how does the message get stored for later delivery and how does User B eventually receive it)?</summary>

**User B is online:**

```
1. User A's client sends via WebSocket:
   { type: "message.send", conversationId: "conv-123", clientMsgId: "client-abc", content: "Hello" }

2. WebSocket Gateway receives the frame, validates the JWT session,
   forwards to Chat Service via internal RPC.

3. Chat Service:
   a. Validates: Is User A a member of conv-123? Is content within limits?
   b. Deduplicates: Checks clientMsgId against a short-lived cache (Redis, 5-min TTL).
      If duplicate → returns existing messageId, skips re-processing.
   c. Assigns messageId (ULID for sortability) and server timestamp.
   d. Writes message to Message Store (Cassandra):
      Partition key: conversationId, Sort key: messageId/timestamp.
   e. Updates conversation's updatedAt for conversation list ordering.
   f. Publishes event to Kafka: { topic: "messages", key: "conv-123", payload: message }

4. Chat Service ACKs back to User A via the gateway:
   { type: "message.sent", clientMsgId: "client-abc", messageId: "msg-456", timestamp: 1710000000 }
   User A's UI updates: single check mark (sent).

5. Chat Service looks up User B's connection in Redis Connection Registry:
   GET user:userB:gateway → "gateway-server-7"

6. Chat Service sends the message to gateway-server-7 via internal RPC.

7. Gateway-server-7 pushes to User B's WebSocket:
   { type: "message.new", messageId: "msg-456", conversationId: "conv-123",
     senderId: "userA", content: "Hello", timestamp: 1710000000 }

8. User B's client renders the message and sends delivery ACK:
   { type: "message.ack", messageId: "msg-456" }

9. Gateway forwards ACK to Chat Service, which notifies User A:
   { type: "message.delivered", messageId: "msg-456" }
   User A's UI updates: double check mark (delivered).
```

**User B is offline:**

Steps 1-4 are identical. At step 5:

```
5. Redis lookup returns no entry for User B (no active connection).

6. Chat Service marks the message as "undelivered" for User B.
   The message is already persisted in the Message Store — no separate
   offline queue needed.

7. Notification Service (consuming from Kafka) detects User B is offline
   (checks Presence Service), sends a push notification via APNs/FCM:
   "User A: Hello"

8. Later, User B opens the app and establishes a WebSocket connection.

9. During connection setup, the client sends its last known message ID
   or timestamp for each conversation.

10. Chat Service queries the Message Store for messages after that point
    and delivers them as a batch:
    { type: "message.sync", conversationId: "conv-123", messages: [...] }

11. Once delivered, the same delivery ACK flow runs (step 8-9 above).
```

The key insight is that there's no separate "offline queue." The Message Store IS the durable queue. The difference between online and offline delivery is just the transport: WebSocket push vs catch-up sync on reconnect.

</details>

<details>
<summary>7. Design the WebSocket gateway layer — how does a client establish a WebSocket connection (including JWT authentication during the handshake), what is the full connection lifecycle (connect, authenticate, subscribe, heartbeat, disconnect), and how should the client handle reconnection after a dropped connection (including catching up on missed messages)?</summary>

**Connection establishment and auth:**

```typescript
// Client-side connection with JWT in the protocol or query param
// Option 1: Token in query string (simpler, but logged in some proxies)
const ws = new WebSocket("wss://chat.example.com/ws?token=eyJhbG...");

// Option 2: Token in Sec-WebSocket-Protocol header (preferred)
const ws = new WebSocket("wss://chat.example.com/ws", [`bearer.${jwt}`]);
```

On the server side:

```typescript
// Gateway handles the HTTP upgrade
server.on("upgrade", async (req, socket, head) => {
  try {
    const token = extractToken(req); // from query or protocol header
    const user = await verifyJWT(token);

    wss.handleUpgrade(req, socket, head, async (ws) => {
      // Register connection in Redis
      await redis.set(`user:${user.id}:gateway`, GATEWAY_ID, "EX", 300);
      // Store in local connection map
      connections.set(user.id, ws);
      wss.emit("connection", ws, req, user);
    });
  } catch (err) {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");
    socket.destroy();
  }
});
```

**Full connection lifecycle:**

```
1. CONNECT:   Client opens WSS connection with JWT.
              Server validates token during HTTP upgrade.
              If invalid → 401, connection rejected.

2. REGISTER:  Server registers (userId → gatewayId) in Redis with TTL (5 min).
              Server adds connection to in-memory map.

3. SYNC:      Client sends last-known state:
              { type: "sync", conversations: { "conv-123": "last-msg-id-789" } }
              Server delivers missed messages since those IDs.

4. HEARTBEAT: Client sends { type: "ping" } every 30 seconds.
              Server responds { type: "pong" }.
              Server refreshes Redis TTL on each ping.
              If no ping received in 90s → server considers client dead,
              closes connection, removes from registry.

5. OPERATE:   Normal message sending/receiving.

6. DISCONNECT: Clean: client sends close frame. Server removes from registry.
               Dirty: TCP drops. Server detects via missed heartbeats (90s timeout).
               Either way: Redis entry removed, presence updated to offline.
```

**Why heartbeats even though TCP has keepalive:**

- TCP keepalive defaults are too slow (2 hours on Linux). Chat needs detection within 1-2 minutes.
- Many mobile networks silently drop idle connections. NAT tables expire after 30-60 seconds of inactivity. Without application-level heartbeats, the server thinks the client is connected but the path is gone.
- Heartbeats also serve as a signal to refresh the Redis registry TTL — if a gateway crashes, stale entries auto-expire rather than requiring active cleanup.

**Reconnection strategy:**

```typescript
class ChatClient {
  private reconnectAttempt = 0;
  private maxDelay = 30_000;
  private lastMessageIds: Map<string, string>; // conversationId → lastMessageId

  private reconnect() {
    // Exponential backoff with jitter
    const base = Math.min(1000 * 2 ** this.reconnectAttempt, this.maxDelay);
    const jitter = Math.random() * base * 0.5;
    const delay = base + jitter;

    setTimeout(() => {
      this.reconnectAttempt++;
      this.connect();
    }, delay);
  }

  private onOpen() {
    this.reconnectAttempt = 0;
    // Send sync request with last known message per conversation
    this.send({
      type: "sync",
      conversations: Object.fromEntries(this.lastMessageIds),
    });
  }
}
```

The jitter prevents thundering herd when a gateway crashes and thousands of clients reconnect simultaneously (covered in detail in question 17).

</details>

<details>
<summary>8. How does the system route a message to the correct WebSocket server when the recipient could be connected to any server in the fleet — design a message routing layer using a centralized connection registry in Redis, explain how pub/sub enables cross-server delivery, and why might you use consistent hashing to assign users to gateway servers instead of (or in addition to) a registry lookup?</summary>

**Approach 1: Centralized Connection Registry (most common)**

```
Redis stores: user:{userId}:gateway → "gateway-server-7"

Message routing flow:
1. Chat Service needs to deliver to User B.
2. GET user:userB:gateway → "gateway-server-7"
3. Chat Service sends message to gateway-server-7 via internal RPC (gRPC).
4. Gateway-server-7 looks up User B in its local connection map, pushes via WebSocket.
```

This is simple and flexible — users can connect to any gateway server, and the registry always knows where they are. Downsides: every message delivery requires a Redis lookup, and the registry must stay consistent (stale entries after crashes).

**Approach 2: Redis Pub/Sub for cross-server delivery**

```
Instead of direct RPC, each gateway subscribes to a Redis channel per server:

- gateway-server-1 subscribes to channel "gateway:1"
- gateway-server-7 subscribes to channel "gateway:7"

Delivery flow:
1. Chat Service looks up User B → gateway-server-7
2. PUBLISH gateway:7 { userId: "userB", message: {...} }
3. Gateway-server-7 receives from its subscription, delivers to User B.
```

This decouples the Chat Service from knowing gateway endpoints directly. The Chat Service only needs to know the gateway ID and publish to the right channel. But Redis pub/sub is fire-and-forget (no delivery guarantee), so you still need the message store as the source of truth.

**For group messages**, pub/sub becomes more efficient:

```
1. Chat Service resolves group members → [userA@gw-1, userB@gw-7, userC@gw-7, userD@gw-3]
2. Batch by gateway: { gw-1: [userA], gw-7: [userB, userC], gw-3: [userD] }
3. PUBLISH to each gateway channel once (not once per user).
```

**Approach 3: Consistent Hashing**

Instead of a registry lookup, hash the userId to deterministically route to a gateway:

```
hash(userId) % N → gateway-server-K
```

Benefits:
- No Redis lookup per message — O(1) routing
- No registry to maintain or clean up
- Predictable distribution

Downsides:
- Users MUST connect to their assigned server (needs client-side or load-balancer-level routing)
- Adding/removing servers rehashes ~1/N users (with consistent hashing ring, not modulo)
- Less flexible — you can't just connect to any server

**In practice, most systems use the connection registry** because the flexibility outweighs the cost of a Redis lookup (~0.5ms). Consistent hashing is used when the fleet is very large (thousands of gateway servers) and the Redis registry becomes a bottleneck, or as an optimization layer on top of the registry — hash to narrow down which cluster of gateways a user might be on, then do a registry lookup within that cluster.

</details>

## Component Deep Dives

<details>
<summary>9. For group messaging, compare fan-out on write (pre-computing each member's inbox at send time) vs fan-out on read (writing once and resolving recipients at read time) — what are the tradeoffs in write amplification, read latency, and storage cost, why do most systems use different strategies for small groups vs large groups (e.g., channels with thousands of members), and where is the typical threshold?</summary>

**Fan-out on write**: When a message is sent to a group, the system writes a copy (or pointer) to each member's inbox at send time.

```
User A sends to group (100 members)
→ Write 100 entries: one per member's inbox
→ Each member reads from their own inbox (fast, single partition read)
```

**Fan-out on read**: The message is written once to the conversation's message stream. When a member opens the conversation, they read from the shared stream.

```
User A sends to group (100 members)
→ Write 1 entry to conversation stream
→ Each member reads from the conversation partition (may need to merge across conversations for a unified inbox view)
```

| Aspect | Fan-out on Write | Fan-out on Read |
|---|---|---|
| **Write cost** | O(N) per message (N = members) | O(1) per message |
| **Read cost** | O(1) per user — read own inbox | O(K) per user — read K conversation streams, merge |
| **Storage** | N copies (or N pointers) per message | 1 copy per message |
| **Latency at send time** | Higher (must fan out to N inboxes) | Lower (single write) |
| **Latency at read time** | Lower (pre-computed inbox) | Higher (assemble on the fly) |
| **Membership changes** | Complex — adding a member means backfilling their inbox | Simple — new member reads from conversation stream from join point |

**Why hybrid (small groups vs large groups):**

- **Small groups (< 100-500 members)**: Fan-out on write works well. Write amplification is bounded (100 writes per message), and the benefit of instant inbox reads is significant for mobile clients that need to show a unified "recent conversations" view quickly.
- **Large groups/channels (500+ members)**: Fan-out on write becomes prohibitive. A single message in a 10K-member channel generating 10K writes would saturate write capacity. Fan-out on read keeps writes at O(1) and reads are acceptable because large channels are typically read less frequently per member (most members don't read every message).

**Typical threshold**: 100-500 members. Slack, Discord, and WhatsApp all use variants of this hybrid approach. WhatsApp caps groups at 1024 members. Slack uses fan-out on write for small channels and fan-out on read (with caching) for large channels.

**Practical optimization**: Even with fan-out on write, you don't copy the full message to each inbox. You write a lightweight pointer `{ messageId, conversationId, timestamp }` to each member's inbox. The actual message content lives in the conversation's message stream.

</details>

<details>
<summary>10. How do you model group roles (admin, member, owner), permissions (who can post, pin messages, add members), and membership changes (joins, leaves, kicks) — what fields and relationships does the data model need, how does the application layer enforce permissions, and what are the tradeoffs of checking permissions at the gateway vs at the chat service?</summary>

**Data model:**

```typescript
// ConversationMember (as introduced in question 3, extended with role details)
{
  conversationId: string,
  userId: string,
  role: "owner" | "admin" | "member",
  joinedAt: number,
  addedBy: string | null,      // who invited this user
  lastReadMessageId: string | null,
  muted: boolean
}

// ConversationSettings (per-conversation permission config)
{
  conversationId: string,
  whoCanPost: "everyone" | "admins_only",       // useful for announcement channels
  whoCanAddMembers: "everyone" | "admins_only",
  whoCanPinMessages: "everyone" | "admins_only",
  whoCanEditInfo: "admins_only",                // group name, avatar
  maxMembers: number
}
```

**Permission hierarchy**: owner > admin > member. Owner can do everything including deleting the group and promoting admins. Admin can manage members and moderate. Member can post and read (unless restricted by settings).

**Application-layer enforcement:**

```typescript
function canPerformAction(
  member: ConversationMember,
  settings: ConversationSettings,
  action: "post" | "add_member" | "pin" | "kick" | "edit_info"
): boolean {
  if (!member) return false; // not a member at all

  switch (action) {
    case "post":
      return settings.whoCanPost === "everyone" || member.role !== "member";
    case "add_member":
      return settings.whoCanAddMembers === "everyone" || member.role !== "member";
    case "pin":
      return settings.whoCanPinMessages === "everyone" || member.role !== "member";
    case "kick":
      return member.role === "owner" || member.role === "admin";
    case "edit_info":
      return member.role === "owner" || member.role === "admin";
  }
}
```

**Membership change events:**

```typescript
// Written to the message stream as system messages
{ type: "system", content: "User X was added by User Y", subtype: "member_added" }
{ type: "system", content: "User X left the group", subtype: "member_left" }
{ type: "system", content: "User X was removed by Admin Y", subtype: "member_kicked" }
{ type: "system", content: "User X is now an admin", subtype: "role_changed" }
```

These system messages appear inline in the conversation, so all members see membership changes in context.

**Where to check permissions — gateway vs chat service:**

| Aspect | At Gateway | At Chat Service |
|---|---|---|
| **Latency** | Faster — rejects before RPC | Adds one hop |
| **Consistency** | May use stale cached data | Source of truth, always current |
| **Complexity** | Duplicates business logic in infra layer | Clean separation of concerns |
| **Security** | Good for coarse checks (is user authenticated?) | Required for fine-grained checks (is user admin of THIS group?) |

**Recommended approach**: Gateway does coarse checks (valid JWT, user exists, rate limiting). Chat Service does fine-grained permission checks (is this user a member of this conversation, do they have the right role for this action). This keeps business logic in one place and avoids stale permission data at the gateway — a user might be kicked from a group, and the gateway's cached membership could be seconds behind.

</details>

<details>
<summary>11. What happens to message delivery when a group member is added or removed mid-conversation — how do you decide which messages the new member can see (full history vs from-join-point), how do you prevent race conditions where a kicked user sends a message before the membership change propagates, and what consistency guarantees does the system need for membership operations?</summary>

**New member — history visibility:**

Two common approaches:

1. **From-join-point only** (WhatsApp's approach): New member sees messages from `joinedAt` onwards. Simpler to implement — the client requests history with `after: joinedAt` and the server filters. Protects privacy of conversations before the member joined.

2. **Full history** (Slack's approach): New member can scroll back through the entire conversation. Better for team contexts where the history is the value. Implemented by simply not filtering — the new member's client fetches from the conversation stream like any other member.

The choice is a product decision. Store `joinedAt` on the membership record either way — even with full history access, you might want to visually distinguish "messages before you joined."

**Race condition: kicked user sends a message**

The timeline of concern:

```
T0: Admin sends "kick User X" request
T1: Chat Service removes User X from membership table
T2: User X's client sends a message (hasn't received kick notification yet)
T3: Chat Service receives User X's message, checks membership → X is no longer a member → REJECT
```

This works if T3 > T1, which is the normal case. The dangerous scenario:

```
T0: Admin sends "kick User X"
T1: User X sends a message (both in flight simultaneously)
T2: Chat Service processes User X's message BEFORE processing the kick → message goes through
T3: Chat Service processes the kick → User X is removed
```

**How to handle this:**

- **Accept the race**: In most chat systems, a message slipping through during the milliseconds between kick initiation and processing is acceptable. The message is valid — the user WAS a member when they sent it.
- **Serialize membership operations**: Use a per-conversation lock (Redis distributed lock or a dedicated membership service that serializes operations per conversation). This ensures the kick is fully processed before any new messages are accepted. This adds latency and complexity.
- **Soft delete + cleanup**: Process the kick, then retroactively delete any messages sent by the user after the kick timestamp. This is eventually consistent but avoids the complexity of serialization.

**Consistency guarantees needed:**

- Membership writes should be **strongly consistent** — after a kick operation returns success, subsequent permission checks must reflect the removal. This rules out eventually-consistent reads from replicas for permission checks.
- The membership change notification to the kicked user's client should be sent **immediately** so their UI updates and stops accepting input.
- For other group members, eventual consistency (sub-second) for membership changes is fine — seeing "User X is typing" for a few hundred milliseconds after a kick is not harmful.

**Practical flow for a kick:**

```
1. Admin sends kick request.
2. Chat Service removes User X from ConversationMember table.
3. Chat Service writes system message: "User X was removed by Admin."
4. Chat Service publishes membership change event.
5. WebSocket Gateway sends kick notification to User X's client.
6. Client-side: remove conversation from UI, close subscription.
7. If User X tries to send a message after step 2, the permission check rejects it.
```

</details>

<details>
<summary>12. Design the message storage layer using a NoSQL database like Cassandra or DynamoDB — how do you choose the partition key and sort key to support the primary access pattern (fetch messages for a conversation in reverse chronological order), how does cursor-based pagination work for chat history, and what are the tradeoffs of partitioning by conversation_id vs by user_id for different read patterns?</summary>

**Primary table design (Cassandra):**

```sql
CREATE TABLE messages (
    conversation_id TEXT,
    message_id TIMEUUID,   -- or ULID, naturally sortable by time
    sender_id TEXT,
    content TEXT,
    content_type TEXT,
    client_msg_id TEXT,
    created_at TIMESTAMP,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

- **Partition key**: `conversation_id` — all messages for a conversation are co-located on the same partition. This matches the primary access pattern: "fetch messages for this conversation."
- **Sort key**: `message_id` (time-sortable ID like ULID or Cassandra's TIMEUUID) — messages are sorted within the partition, enabling efficient range scans.
- **Descending order**: Most recent messages first, since users typically load the latest messages and scroll up.

**DynamoDB equivalent:**

```
Table: Messages
  PK: conversationId (String)
  SK: messageId (String, ULID for lexicographic time sort)
  GSI: userId-timestamp-index (for "all messages by user" queries if needed)
```

**Cursor-based pagination:**

```typescript
// Client requests: GET /conversations/conv-123/messages?cursor=01ARZ3NDEKTSV4RRFFQ69G5FAV&limit=50
// cursor = last messageId from previous page (or omitted for first page)

// Cassandra query:
// First page (no cursor):
SELECT * FROM messages
WHERE conversation_id = 'conv-123'
ORDER BY message_id DESC
LIMIT 50;

// Subsequent pages:
SELECT * FROM messages
WHERE conversation_id = 'conv-123'
AND message_id < cursor_message_id  -- "older than last seen"
ORDER BY message_id DESC
LIMIT 50;
```

The response includes a `nextCursor` (the last messageId in the result set) for the client to use on the next request. If fewer than `limit` results are returned, there are no more pages.

**Why cursor-based over offset-based**: Offset pagination (`SKIP 100 LIMIT 50`) is unreliable when new messages are being inserted. A message inserted between page fetches causes duplicates or missed messages. Cursor-based is stable because it anchors to a specific point in the stream.

**Partitioning by conversation_id vs user_id:**

| Approach | Optimizes for | Downside |
|---|---|---|
| **conversation_id** (recommended) | Fetching conversation history — single partition read | Building a unified inbox ("all my conversations, sorted by latest message") requires reading multiple partitions |
| **user_id** | Unified inbox view — single partition read for all of a user's messages | Fetching a specific conversation requires filtering across the user's entire message stream |

**Recommended**: Partition by `conversation_id` for the messages table (the primary access pattern is conversation history), and maintain a separate **inbox table** for the unified conversation list:

```sql
CREATE TABLE user_inbox (
    user_id TEXT,
    updated_at TIMESTAMP,
    conversation_id TEXT,
    last_message_preview TEXT,
    unread_count INT,
    PRIMARY KEY (user_id, updated_at)
) WITH CLUSTERING ORDER BY (updated_at DESC);
```

This denormalized inbox table is updated on each message (or via fan-out on write for small groups) and gives instant "recent conversations" queries.

**Hot partition concern**: A very active group conversation could create a hot partition. Mitigation: for extremely large groups, bucket the partition by time (`conversation_id#2024-03`) so old messages are in separate partitions. This adds complexity to queries that cross bucket boundaries but prevents unbounded partition growth.

</details>

<details>
<summary>13. How do you ensure messages appear in the correct order within a conversation — why can't you rely solely on client timestamps, how do server-assigned timestamps or per-conversation sequence numbers solve this, what happens when two messages arrive at the server simultaneously, and how does ordering become harder in distributed or multi-region setups where multiple servers handle the same conversation?</summary>

**Why client timestamps fail:**

- Clients have clock skew — a phone 5 minutes behind will produce timestamps that place messages in the wrong position
- Malicious clients can set arbitrary timestamps
- Even with NTP, clocks across millions of devices drift by seconds

**Solution 1: Server-assigned timestamps**

The server stamps each message with its own clock when it receives the message. This works well for a single server, but:

- If two messages arrive at the same millisecond, you get a tie. Break ties with a secondary sort (e.g., messageId which is a ULID with random suffix).
- Server clock is the authority, so even if User A sent before User B on the client side, the server arrival order wins. This is acceptable — "order" in a distributed system means the order the system observes, not the order users intended.

**Solution 2: Per-conversation sequence numbers (stronger guarantee)**

```typescript
// Atomic increment per conversation
async function assignSequenceNumber(conversationId: string): Promise<number> {
  // Redis INCR is atomic — no two messages get the same number
  return await redis.incr(`seq:${conversationId}`);
}

// Message stored with both:
{
  messageId: "01ARZ3...",       // globally unique, time-sortable
  sequenceNum: 4523,            // per-conversation, gap-free, total order
  serverTimestamp: 1710000000   // for display purposes
}
```

Sequence numbers give a **total order** within a conversation — no ties, no ambiguity. The client can also detect gaps (received seq 4523, then 4525 — seq 4524 is missing, request it).

**Simultaneous messages:**

With sequence numbers, the Redis INCR serializes them — one gets 4523, the other gets 4524. The "first" one is whichever the server processes first. This is deterministic and consistent across all clients.

**Multi-region / distributed challenges:**

When multiple servers in different regions can accept messages for the same conversation:

- **Single-leader per conversation**: Route all messages for a conversation to a single region/server. That server assigns sequence numbers. Downside: latency for users far from the leader, and failover complexity.
- **Lamport timestamps / vector clocks**: Each server maintains a logical clock. Messages carry the clock value. This gives partial ordering (causality) but not total ordering — two concurrent messages might not have a defined order.
- **Hybrid Logical Clocks (HLC)**: Combine physical timestamps with logical counters. Gives causal ordering with near-real-time timestamps. Used by CockroachDB and some distributed systems.

**Practical recommendation**: For most chat systems, use a single sequence number counter per conversation (Redis INCR or a dedicated sequencing service). Route all messages for a conversation through the same sequencing point. The latency cost (one Redis round-trip, ~1ms) is negligible compared to network latency to the user. Multi-region setups can use a per-region leader with sequence number ranges (region A gets even numbers, region B gets odd) to avoid coordination, at the cost of non-contiguous sequences.

</details>

<details>
<summary>14. Design the delivery guarantee system that tracks message states (sent, delivered, read) — how does the server confirm delivery to the client, why do you need at-least-once delivery semantics with idempotency keys and client-side deduplication rather than exactly-once, and what is the concrete flow when a message is sent but the ACK is lost (how does the client retry without the recipient seeing duplicates)?</summary>

**Message states:**

```
SENT       → Server has accepted and persisted the message (single check ✓)
DELIVERED  → Recipient's client has received the message (double check ✓✓)
READ       → Recipient has viewed the message (blue double check ✓✓)
```

**Server ACK flow (confirming "sent"):**

```
1. Client sends: { type: "message.send", clientMsgId: "abc-123", ... }
2. Server persists message, assigns messageId.
3. Server replies: { type: "message.sent", clientMsgId: "abc-123", messageId: "msg-456" }
4. Client matches by clientMsgId, updates local message state to "sent."
```

**Why at-least-once, not exactly-once:**

Exactly-once delivery is impossible in a distributed system with unreliable networks (this is a fundamental result from distributed systems theory). The network can always drop a message or an ACK, and the sender can't distinguish "message was lost" from "message was delivered but ACK was lost." So the sender must retry, which means the recipient might get the same message twice. The system must be designed to handle duplicates.

**Idempotency and deduplication flow:**

```
Scenario: Client sends message, server persists it, but the ACK is lost.

1. Client sends: { clientMsgId: "abc-123", content: "Hello" }
2. Server receives, persists as msg-456, sends ACK.
3. ACK is lost (network drop, gateway restart, etc.)
4. Client doesn't receive ACK within timeout (e.g., 5 seconds).
5. Client retries: { clientMsgId: "abc-123", content: "Hello" }  ← same clientMsgId
6. Server receives, checks dedup cache:
   Redis: GET dedup:abc-123 → "msg-456" (exists!)
7. Server returns existing ACK: { clientMsgId: "abc-123", messageId: "msg-456" }
   Does NOT create a second message.
```

**Server-side deduplication implementation:**

```typescript
async function processMessage(msg: IncomingMessage): Promise<string> {
  // Check dedup cache (short TTL — 5 minutes is enough for retry windows)
  const existing = await redis.get(`dedup:${msg.clientMsgId}`);
  if (existing) {
    return existing; // return existing messageId, skip processing
  }

  const messageId = generateULID();
  await messageStore.write({ ...msg, messageId });

  // Cache for deduplication (SET NX to handle concurrent retries)
  await redis.set(`dedup:${msg.clientMsgId}`, messageId, "EX", 300, "NX");

  return messageId;
}
```

**Client-side deduplication for the recipient:**

Even with server-side dedup, the recipient might receive the same message twice (e.g., the server pushes the message, the client ACK is lost, the server retries push). The client maintains a local set of recently seen messageIds:

```typescript
const recentMessageIds = new Set<string>(); // or LRU cache

function onMessageReceived(msg: Message) {
  if (recentMessageIds.has(msg.messageId)) return; // duplicate, ignore
  recentMessageIds.add(msg.messageId);
  renderMessage(msg);
  sendDeliveryAck(msg.messageId);
}
```

**Delivery and read receipt flow:**

```
DELIVERED:
1. Recipient client receives message.new, renders it.
2. Client sends: { type: "message.ack", messageId: "msg-456" }
3. Server updates message state to DELIVERED.
4. Server notifies sender: { type: "message.delivered", messageId: "msg-456" }

READ:
1. Recipient opens conversation and views message.
2. Client sends: { type: "message.read", conversationId: "conv-123", messageId: "msg-456" }
3. Server updates read position (as covered in question 15).
4. Server notifies sender: { type: "message.read", messageId: "msg-456" }
```

</details>

<details>
<summary>15. Design the read receipt and unread count features — what events flow between clients and the server when a user reads a message, how do you track per-user read positions within a conversation efficiently (without writing a row per user per message), how do unread counts get calculated from read positions, and how do these features degrade in large groups where fan-out of read receipts becomes expensive?</summary>

**Core design: per-user read position, not per-message read state.**

Instead of storing "User B read message 4523, 4524, 4525..." (one row per user per message), store a single high-water mark: "User B has read up to message 4525 in conversation conv-123." This is the `lastReadMessageId` on the `ConversationMember` row (as designed in question 3).

**Event flow when User B reads messages:**

```
1. User B opens conversation conv-123.
   The newest visible message is msg-4525.

2. Client sends (debounced — wait 500ms after scrolling stops):
   { type: "message.read", conversationId: "conv-123", upTo: "msg-4525" }

3. Server updates ConversationMember:
   SET lastReadMessageId = "msg-4525", lastReadAt = now()
   WHERE conversationId = "conv-123" AND userId = "userB"
   (Only update if msg-4525 > current lastReadMessageId — never move backward)

4. Server notifies sender(s):
   For 1:1: { type: "message.read", conversationId: "conv-123",
              userId: "userB", readUpTo: "msg-4525" }
   Sender's client marks all messages up to 4525 as "read."
```

**Unread count calculation:**

```typescript
// Unread count = messages in conversation after user's read position
// Option 1: Count query (fine for small conversations)
SELECT COUNT(*) FROM messages
WHERE conversation_id = 'conv-123'
AND message_id > last_read_message_id;

// Option 2: Maintain a counter (better at scale)
// Each new message increments unread_count for all members except the sender.
// Reading resets it: unread_count = total_messages - read_position.
// Store on the inbox table:
{
  userId: "userB",
  conversationId: "conv-123",
  unreadCount: 3,              // pre-computed
  lastReadMessageId: "msg-4525"
}
```

Option 2 (pre-computed counter on the inbox table) is preferred because the "conversation list with unread counts" query is extremely frequent (every time the app opens) and must be fast.

**Typing indicators:**

```
1. Client detects keystroke in conversation input.
2. Client sends (throttled — max once per 3 seconds):
   { type: "typing.start", conversationId: "conv-123" }
3. Server broadcasts to other online members of conv-123.
4. Recipients show "User B is typing..." for 4 seconds (auto-expire).
5. If User B stops typing, client sends:
   { type: "typing.stop", conversationId: "conv-123" }
```

Typing indicators are ephemeral — never persisted, fire-and-forget delivery, no ACK needed.

**Degradation in large groups:**

| Feature | Small group (< 100) | Large group (1000+) |
|---|---|---|
| **Read receipts** | Fan out to all members — show "read by 5 of 8" | Don't fan out individual read receipts. Show only aggregate: "read by 847." Compute lazily on request, not on every read event. |
| **Unread counts** | Pre-computed counter, updated on each message | Same approach works — increment counter for each member. But fan-out on write of the counter update is expensive. Use async batch updates via Kafka. |
| **Typing indicators** | Broadcast to all members | Broadcast only to members who currently have the conversation open (tracked via "subscribe" events). Or suppress entirely — "47 people are typing" is not useful. |

**Throttling strategies for large groups:**

- Batch read receipt updates: instead of processing each read event individually, accumulate them and update aggregate counts every few seconds
- Rate-limit typing indicator broadcasts: one per user per 5 seconds in large groups
- Client-side: only send read receipts for the conversation the user is actively viewing, not for all conversations

</details>

## Scaling & Failure Handling

<details>
<summary>16. How do you scale a fleet of WebSocket gateway servers — why is scaling stateful WebSocket connections fundamentally different from scaling stateless HTTP services, how does graceful connection draining work during deployments (move connections without dropping messages), and what are the tradeoffs of sticky sessions (routing a user to the same server) vs a connection registry (any server can route to any user)?</summary>

This is a common follow-up because getting it wrong means every deployment drops active users.

**Why WebSocket scaling is fundamentally different:**

Stateless HTTP services can be scaled by adding servers behind a load balancer — any server handles any request, and a server can be removed instantly because no request depends on it being there. WebSocket connections are **stateful and long-lived** (hours to days). Each connection holds in-memory state: the user's identity, subscriptions, and the TCP/TLS session. You can't just kill a server — you must migrate or reconnect every user on it.

Key differences:
- **Memory per connection**: Each WebSocket connection uses ~20-50 KB of server memory (TCP buffers, TLS state, application state). A server with 16 GB of available memory can hold ~100K-500K connections.
- **CPU**: Mostly idle — spikes on message bursts. CPU is rarely the bottleneck.
- **Scale-up signal**: Monitor connection count per server, not request rate. Add servers when connection count approaches the threshold.

**Graceful connection draining during deployments:**

```
1. Mark server as "draining" — stop accepting new WebSocket upgrades.
   Load balancer removes it from the pool for new connections.

2. Continue serving existing connections normally (messages flow in/out).

3. Send "reconnect" signal to connected clients in batches:
   { type: "server.reconnect", reason: "maintenance", delay: random(0, 30000) }
   The random delay prevents thundering herd.

4. Clients disconnect and reconnect to a different gateway
   (load balancer routes them to a healthy server).

5. After all connections have drained (or after a timeout, e.g., 60s),
   shut down the server.

6. During the drain window, messages for migrating users are buffered
   or routed to the new gateway via the connection registry
   (which updates as clients reconnect).
```

**Sticky sessions vs connection registry:**

| Aspect | Sticky Sessions | Connection Registry |
|---|---|---|
| **How it works** | Load balancer routes a userId to the same server (hash-based or cookie-based) | Any server can hold any user. Redis stores user → server mapping. |
| **Routing messages** | Hash userId to find their server directly | Look up in Redis, then route to that server |
| **Adding servers** | Rehashes some users to new servers — forces reconnection | New server joins the pool. Only new connections go there. |
| **Removing servers** | All users on that server must reconnect and may land on a different server | Same, but the registry automatically updates as they reconnect |
| **Failover** | If a server dies, its users are rehashed to other servers. Load is unpredictable. | If a server dies, its users reconnect to any available server. Load distributes naturally. |
| **Extra infrastructure** | None — the load balancer handles routing | Redis cluster for the registry |
| **Operational complexity** | Lower | Slightly higher (must manage registry consistency, TTLs, stale cleanup) |

**Recommendation**: Use a connection registry. The flexibility of decoupling user-to-server assignment from the load balancer is worth the Redis overhead. Sticky sessions create uneven load distribution (some users are far more active than others), and server removal is disruptive. The registry approach lets you drain and replace servers gracefully without affecting the rest of the fleet.

</details>

<details>
<summary>17. A WebSocket gateway server crashes and 50,000 users lose their connections simultaneously — walk through what happens: how do clients detect the failure and reconnect, how does the connection registry get cleaned up (stale entries), how do messages sent during the outage get buffered and delivered, and what design choices prevent this from cascading into a "thundering herd" that crashes other gateway servers?</summary>

**Step-by-step recovery:**

**1. Clients detect the failure:**

- The TCP connection drops — the client's WebSocket `onclose` or `onerror` event fires immediately (if the OS detects the RST/FIN) or after the heartbeat timeout (if the connection silently dies, e.g., the server process crashes without closing sockets).
- Heartbeat detection: client sent a ping 30s ago, no pong received within 10s — client considers the connection dead.

**2. Clients reconnect with exponential backoff + jitter:**

Using the backoff strategy from question 7 (exponential backoff with jitter capped at 30s), 50,000 clients spread their reconnections over ~10-15 seconds instead of a single spike.

**3. Connection registry cleanup:**

The crashed server's 50K entries in Redis (`user:{userId}:gateway → "crashed-server"`) are now stale. Three cleanup mechanisms:

- **TTL expiry**: Every registry entry has a 5-minute TTL, refreshed by heartbeats. Since the server is dead, no refreshes happen. Entries auto-expire within 5 minutes.
- **Overwrite on reconnect**: When a client reconnects to a new server, the new server writes `SET user:{userId}:gateway "new-server"`, overwriting the stale entry immediately. This is the fast path — most entries are cleaned up within seconds as clients reconnect.
- **Active cleanup (optional)**: A health checker detects the dead server and bulk-deletes its registry entries: `SCAN` for entries pointing to the dead server and delete them. This handles users who don't reconnect quickly (app backgrounded, phone offline).

**4. Message delivery during the outage window:**

```
Messages sent TO these 50K users between crash and reconnection:

1. Chat Service looks up User B in registry → finds "crashed-server" (stale entry).
2. Chat Service tries to deliver to crashed-server → RPC fails (connection refused).
3. Chat Service marks delivery as failed for this attempt.
   The message is already persisted in the Message Store — it's safe.
4. When User B reconnects, the sync phase delivers all missed messages
   (client sends last known messageId, server returns everything after that).
5. For urgency, the Notification Service sends a push notification
   (it checks presence, finds the user offline, triggers APNs/FCM).
```

No separate buffer is needed — the Message Store acts as the durable buffer.

**5. Preventing cascading failure (thundering herd):**

| Design Choice | Why It Helps |
|---|---|
| **Client-side jitter** | Spreads 50K reconnections over 10-15 seconds instead of a single spike |
| **Connection rate limiting on gateways** | Each gateway caps new connections/second (e.g., 500/s). Excess connections get a "retry later" response with a `Retry-After` header. |
| **Load balancer connection limits** | If any single gateway hits its connection threshold, the LB stops routing new upgrades to it. |
| **Over-provisioned fleet** | Run at 60-70% capacity so the remaining servers can absorb the displaced connections without hitting limits. |
| **Circuit breaker on registry lookups** | If Redis is being hammered by 50K simultaneous SET operations, batch them or use pipelining. |

**Capacity math**: 50K users reconnecting across a fleet of, say, 20 remaining servers = 2,500 new connections per server. With jitter spreading over 10 seconds, that's ~250 new connections/second per server — well within normal capacity.

</details>

<details>
<summary>18. The message store (e.g., Cassandra cluster) becomes temporarily unavailable — how does the system degrade gracefully so users can still send and receive real-time messages even if chat history is inaccessible, what buffering or write-ahead strategies prevent message loss during the outage, and how do you reconcile buffered messages once the store recovers?</summary>

**Graceful degradation — what still works:**

The key insight is that real-time message delivery (WebSocket push) does not require the message store. The flow is: Chat Service receives message → pushes to recipient via gateway. The store write can be decoupled.

| Feature | During Outage | Why |
|---|---|---|
| **Sending/receiving real-time messages** | Works (degraded) | Messages route through WebSocket gateways. The delivery path doesn't block on the store write. |
| **Chat history / scroll-back** | Unavailable | History reads go to the store, which is down. Client shows "History temporarily unavailable." |
| **Search** | Unavailable | Search indexes are fed from the store. |
| **Unread counts** | Stale | Counts are computed from the store or inbox table. May show incorrect numbers until recovery. |
| **Offline user delivery** | Delayed | Messages for offline users rely on the sync phase (which reads from the store). They'll get messages after recovery. Push notifications still work. |

**Write-ahead buffering to prevent message loss:**

```
Normal flow:
  Chat Service → Write to Cassandra → ACK to sender → Deliver to recipient

Degraded flow (store unavailable):
  Chat Service → Write to Kafka (write-ahead log) → ACK to sender → Deliver to recipient
                        ↓
              [Kafka retains messages durably until consumed]
```

Implementation:

```typescript
async function processMessage(msg: ValidatedMessage): Promise<string> {
  const messageId = generateULID();
  const enrichedMsg = { ...msg, messageId, serverTimestamp: Date.now() };

  // Always write to Kafka first (durable log)
  await kafka.produce("messages", {
    key: msg.conversationId,
    value: enrichedMsg,
  });

  // Attempt store write (non-blocking for real-time delivery)
  try {
    await messageStore.write(enrichedMsg);
  } catch (err) {
    if (isStoreUnavailable(err)) {
      // Message is safe in Kafka. Store write will be retried by consumer.
      metrics.increment("message.store_write_deferred");
    } else {
      throw err;
    }
  }

  // Deliver in real-time regardless of store status
  await deliverToRecipients(enrichedMsg);

  return messageId;
}
```

**Alternatively**, if you don't already use Kafka in the write path, use a local write-ahead log:

- Chat Service writes to a local append-only file or an embedded queue (like Redis Streams on the same node).
- A background worker drains the WAL into Cassandra when it recovers.
- Risk: if the Chat Service node itself dies, the local WAL is lost. Kafka is safer because it's a separate distributed durable log.

**Reconciliation after recovery:**

```
1. Cassandra comes back online.
2. Kafka consumer (or WAL drainer) replays buffered messages:
   - For each message, check if it already exists (idempotent write using messageId).
   - Write to Cassandra.
   - Update inbox table, unread counts.
3. Lag metric: monitor Kafka consumer lag to track reconciliation progress.
4. Once lag reaches zero, all buffered messages are persisted.
5. Clients that had "History unavailable" can now load history normally.
```

**Important safeguards:**

- **Ordering**: Kafka partitions by `conversationId`, so messages for the same conversation are replayed in order.
- **Idempotency**: Use `messageId` as the primary key in Cassandra. Re-inserting the same messageId is a no-op (Cassandra upserts by default).
- **Monitoring**: Alert on store unavailability immediately. If the outage lasts longer than Kafka's retention period (e.g., 7 days), messages could be lost. Size Kafka retention to outlast any realistic outage.
- **Client communication**: During degradation, the client UI can show a subtle indicator: "Some features may be limited" — but keep the core chat flowing.

</details>
