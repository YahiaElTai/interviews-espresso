# System Design: News Feed

> **19 questions**

- Requirements and estimation: feed reads vs post writes ratio, fan-out write amplification per post, storage and throughput estimates
- API design: publish post, fetch feed, follow/unfollow, cursor-based pagination
- Data models: users, posts, follows, feed entries, social graph
- High-level architecture: post service, fan-out service, feed service, social graph service
- Fan-out on write (push) vs fan-out on read (pull): latency, storage, consistency tradeoffs
- Hybrid fan-out: follower count threshold, merging pushed and pulled content at read time
- Celebrity problem (hot key / fan-out explosion): why pure push breaks, hybrid solution
- Feed ranking: signals (recency, engagement, affinity), relevance scoring, write-time vs read-time ranking tradeoffs
- Real-time feed updates: polling vs long polling vs SSE vs WebSockets, cost of push at scale
- Fan-out service: follower lookup, batched writes, worker distribution, partial failure handling
- Feed cache: Redis data structure (sorted set, list), post IDs vs full objects, sizing and eviction
- Feed read path: cache lookup, celebrity post merge, post hydration, ranking, cursor-based pagination, cache miss
- Post updates and deletes: propagation through fan-out-on-write without re-fanning, consistency guarantees, tombstones
- Scaling fan-out: viral posts, worker scaling, queue depth, backpressure, active vs inactive user priority
- Partitioning and sharding: feed cache by user ID, post storage by post ID, hot shard handling
- Cache failure: preventing database stampede, request coalescing, circuit breaking, degraded feed

---

## Requirements & Estimation

<details>
<summary>1. How would you scope the requirements for a news feed system — what are the key read vs write ratios, how do you estimate the fan-out work per post, and why does the read-heavy nature of feeds fundamentally shape every architecture decision that follows?</summary>

**Functional requirements:**
- Users can publish posts (text, images, links)
- Users can follow/unfollow other users
- Users see a chronological or ranked feed of posts from accounts they follow
- Feed supports pagination (infinite scroll)

**Non-functional requirements:**
- Feed read latency under 200ms at p99
- Eventual consistency is acceptable (posts appearing within seconds, not real-time)
- High availability — feed should always return something, even if slightly stale

**Key ratios and estimates (for a Twitter/Instagram-scale system):**

| Metric | Estimate |
|---|---|
| DAU | 300M |
| Avg follows per user | 200 |
| Posts per day | ~500M |
| Feed reads per day | ~5B (each user opens feed ~15 times/day) |
| Read:write ratio | ~10:1 |
| Avg followers per poster | 200 |

**Fan-out work per post:** When a user with 200 followers publishes, you need to write that post reference to 200 individual feed caches. A celebrity with 10M followers means 10M writes per post. Across all posts per day, fan-out writes could be 500M posts x 200 avg followers = 100B write operations/day — this is the write amplification problem.

**Why read-heavy shapes everything:** With a 10:1 read-to-write ratio (and feed reads being latency-sensitive), you must pre-compute feeds rather than assembling them on the fly. This drives the fan-out-on-write architecture: pay the cost at write time (which can be async and tolerates latency) so reads are a single cache lookup. Storage decisions, caching strategy, and the entire push vs pull debate all stem from this asymmetry.

**Storage estimates:**
- Each feed entry is ~100 bytes (post ID + metadata)
- 300M users x 500 cached posts = 15TB of feed cache
- Post storage: 500M posts/day x 1KB avg = 500GB/day raw, ~180TB/year

</details>

<details>
<summary>2. Design the API for a news feed system covering publish post, fetch feed, and follow/unfollow — why is cursor-based pagination essential for feeds instead of offset-based, what fields does each endpoint need, and how do you handle the edge cases (empty feeds, deleted posts in a page, real-time insertions shifting the cursor)?</summary>

**Core endpoints:**

```
POST /v1/posts
Body: { content: string, mediaUrls?: string[] }
Response: { postId, createdAt, author }

GET /v1/feed?cursor={cursor}&limit=20
Response: { posts: Post[], nextCursor: string | null }

POST /v1/users/{userId}/follow
DELETE /v1/users/{userId}/follow
Response: { success: boolean }
```

All endpoints require authentication via `Authorization` header. The user ID comes from the auth token, never from the request body (prevents impersonation).

**Why cursor-based pagination is essential:**

Offset-based (`?page=2&limit=20`) breaks in feeds because the dataset is constantly changing. If 5 new posts are inserted while you're on page 1, requesting page 2 with `OFFSET 20` skips none of the old posts but shifts everything — you either see duplicates or miss posts.

Cursor-based pagination uses a stable reference point (typically the last post's timestamp or a composite `timestamp:postId` cursor) so the query is always "give me 20 posts older than this cursor." New insertions at the top don't affect what comes after the cursor.

```typescript
// Cursor is an opaque base64-encoded string
// Internally: { timestamp: number, postId: string }
const decodeCursor = (cursor: string) =>
  JSON.parse(Buffer.from(cursor, 'base64url').toString());

// Query: "posts with score < cursor_score, LIMIT 21"
// Fetch limit+1 to determine if there's a next page
```

**Edge cases:**

- **Empty feed:** Return `{ posts: [], nextCursor: null }`. The client shows an onboarding prompt ("Follow accounts to see posts").
- **Deleted posts in a page:** Filter out deleted/hidden posts after hydration but before returning. If filtering reduces the page below the requested limit, you can either return a short page (simpler) or fetch more to fill the gap (better UX). Short pages are fine — clients handle variable-length responses.
- **Real-time insertions:** New posts get higher scores (timestamps) than the cursor, so they only appear when the user scrolls back to the top or refreshes. The cursor guarantees stable forward pagination. The client can show a "New posts available" banner when it detects new content via polling or push.

</details>

<details>
<summary>3. What data models does a news feed system need — design the schemas for users, posts, follows, feed entries, and the social graph, explaining why the follow relationship and feed entry table are separate from the posts table, and how the social graph structure affects fan-out performance?</summary>

**Core schemas:**

```sql
-- Users
CREATE TABLE users (
  user_id     UUID PRIMARY KEY,
  username    VARCHAR(50) UNIQUE NOT NULL,
  follower_count  INT DEFAULT 0,
  following_count INT DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL
);

-- Posts (source of truth)
CREATE TABLE posts (
  post_id     UUID PRIMARY KEY,
  author_id   UUID NOT NULL REFERENCES users(user_id),
  content     TEXT NOT NULL,
  media_urls  TEXT[],
  is_deleted  BOOLEAN DEFAULT FALSE,
  created_at  TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_posts_author ON posts(author_id, created_at DESC);

-- Follows (social graph)
CREATE TABLE follows (
  follower_id  UUID NOT NULL REFERENCES users(user_id),
  followee_id  UUID NOT NULL REFERENCES users(user_id),
  created_at   TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_follows_followee ON follows(followee_id);
-- followee_id index: "who follows user X?" — needed for fan-out

-- Feed entries (fan-out-on-write materialized feed)
CREATE TABLE feed_entries (
  user_id     UUID NOT NULL,
  post_id     UUID NOT NULL,
  author_id   UUID NOT NULL,
  score       DOUBLE PRECISION NOT NULL, -- ranking score or timestamp
  created_at  TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (user_id, score, post_id)
);
```

In practice, feed entries live in Redis rather than a relational DB (covered in question 11), but the logical model is the same.

**Why separate tables:**

- **Follows separate from posts:** The follow relationship is the social graph — it answers "who should see this post?" It's a many-to-many relationship between users that changes independently of posts. Embedding it in the posts table would mean duplicating follow data per post or denormalizing in ways that make unfollows extremely expensive.

- **Feed entries separate from posts:** The feed entry table is a materialized view — it maps "user X should see post Y." A single post creates N feed entries (one per follower). If you stored this in the posts table, every feed read would require joining posts with follows, which is O(follows) per read. Pre-materializing moves that cost to write time.

**Social graph and fan-out performance:**

The fan-out service needs to answer "who are user X's followers?" efficiently. The `follows` table indexed on `followee_id` provides this, but for users with millions of followers, even this query is expensive. At scale, the social graph is often stored in a dedicated graph database or cached in memory, with follower lists partitioned and streamed to fan-out workers in batches rather than fetched all at once.

The `follower_count` on the user record is critical for the hybrid fan-out decision — it determines whether a post gets pushed (low follower count) or pulled at read time (high follower count) without having to count followers on every post.

</details>

## High-Level Architecture

<details>
<summary>4. Walk through the high-level architecture of a news feed system — what are the responsibilities of the post service, fan-out service, feed service, and social graph service, why do you separate them instead of building a monolith, and how does a post flow from creation to appearing in a follower's feed?</summary>

**Service responsibilities:**

| Service | Responsibility |
|---|---|
| **Post Service** | CRUD for posts. Stores the canonical post data. Handles media upload coordination. Publishes "post created" events. |
| **Social Graph Service** | Manages follow/unfollow relationships. Answers "who follows user X?" and "who does user X follow?" Maintains follower counts. |
| **Fan-out Service** | Consumes "post created" events. Looks up the author's followers via the social graph service. Writes the post reference to each follower's feed cache. Handles batching, retries, and prioritization. |
| **Feed Service** | Handles feed read requests. Reads from the user's feed cache, merges celebrity posts (pull-based), hydrates post IDs into full objects, ranks, filters, and paginates. |

**Supporting infrastructure:**
- **Message queue** (Kafka/SQS) between Post Service and Fan-out Service for async decoupling
- **Redis cluster** for feed caches (sorted sets of post IDs per user)
- **Post cache** (Redis/Memcached) for frequently accessed post objects
- **CDN** for media assets

**Why separate services:**

Each service scales independently along different axes. The fan-out service needs to scale horizontally with worker count during viral posts. The feed service scales with read traffic (which peaks at different times). The social graph service has its own access patterns (burst of follows during events). A monolith would force you to scale everything together and couple deployment cycles.

**Post flow — creation to appearing in a follower's feed:**

1. User calls `POST /v1/posts` → **Post Service** validates, stores the post in the database, returns success to the user immediately
2. Post Service publishes a `PostCreated` event to the message queue with `{ postId, authorId, createdAt }`
3. **Fan-out Service** consumes the event, queries the **Social Graph Service** for the author's follower list
4. Fan-out Service checks the author's follower count:
   - Below threshold (e.g., <100K): push to all followers' feed caches
   - Above threshold: skip fan-out; this post will be pulled at read time
5. For push: Fan-out Service batches followers and writes `(postId, score)` to each follower's **Redis sorted set** via pipeline commands
6. When a follower opens their feed, **Feed Service** reads their sorted set from Redis, merges any pull-based celebrity posts, hydrates post IDs into full post objects (via Post Service/cache), ranks, and returns the paginated response

The user who posted gets an immediate response at step 1. Steps 2-5 happen asynchronously — followers see the post within seconds, not instantly.

</details>

<details>
<summary>5. Explain the fundamental tradeoff between fan-out on write (push) and fan-out on read (pull) — how does each approach work, what are the latency, storage, consistency, and write amplification tradeoffs, and under what conditions does each model break down?</summary>

**Fan-out on write (push):**

When a user publishes a post, immediately write that post reference to every follower's feed cache. The feed is pre-computed — reading is just a cache lookup.

- **Read latency:** Excellent. Single Redis read, O(1) lookup.
- **Write cost:** High. One post = N writes where N is follower count. A post from a user with 1M followers triggers 1M write operations.
- **Storage:** High. Every post is duplicated across all followers' feed caches.
- **Consistency:** Eventual — some followers see the post before others as fan-out progresses.
- **Breaks down when:** A user has millions of followers. Fan-out takes minutes, consumes massive queue/worker capacity, and delays fan-out for everyone else in the queue.

**Fan-out on read (pull):**

When a user opens their feed, query all the accounts they follow, fetch their recent posts, merge, rank, and return. Nothing is pre-computed.

- **Read latency:** Poor. Must query N followed accounts, merge-sort results. Even with caching, this is O(following_count) per read.
- **Write cost:** Minimal. Publishing a post is just one database write.
- **Storage:** Low. No duplication — posts stored once.
- **Consistency:** Strong — you always read the latest state.
- **Breaks down when:** Users follow hundreds of accounts and read frequently. The merge operation is expensive and doesn't cache well because every user follows a different set of accounts.

**Summary table:**

| Dimension | Push (fan-out on write) | Pull (fan-out on read) |
|---|---|---|
| Read latency | Very low (cache hit) | High (multi-source merge) |
| Write amplification | High (1 post → N writes) | None |
| Storage | High (duplicated per follower) | Low (single copy) |
| Celebrity handling | Terrible | Fine |
| Inactive user waste | Writes to users who never read | No wasted work |

Neither model works alone at scale. Push fails for celebrities, pull fails for read-heavy normal users. This is why every production system uses a hybrid (covered in question 6).

</details>

<details>
<summary>6. Why does a hybrid fan-out approach exist and how does it work — what follower count threshold triggers the switch from push to pull, how do you merge pushed and pulled content at read time, and what complexity does the hybrid model add that pure push or pure pull avoids?</summary>

**Why hybrid exists:** Pure push breaks on celebrities (10M writes per post). Pure pull breaks on read latency (merging hundreds of sources per read). Hybrid gets the best of both: push for normal users (fast reads) and pull for celebrities (avoiding write explosion).

**How it works:**

1. When a post is created, check the author's follower count against a threshold
2. **Below threshold** (e.g., <100K followers): fan-out on write — push to all followers' feed caches
3. **Above threshold**: store the post but skip fan-out. Mark the author as a "pull source."
4. At read time, the feed service fetches the user's pre-computed feed (pushed posts) AND queries recent posts from any pull-source accounts the user follows, then merges and ranks the combined set

**Threshold selection:** The exact number varies by system. Twitter reportedly used ~5,000-10,000 followers as the threshold. Instagram and Facebook use more nuanced signals. The threshold is a tuning knob — lower means less fan-out work but more read-time merging. In practice, only ~0.1-1% of users exceed the threshold, but they generate a disproportionate share of fan-out work.

**Read-time merge logic:**

```typescript
async function getFeed(userId: string, cursor: string, limit: number) {
  // 1. Get pre-computed feed (pushed posts)
  const pushedPostIds = await redis.zrevrangebyscore(
    `feed:${userId}`, cursor, '-inf', 'LIMIT', 0, limit + 10 // over-fetch for merge, newest-first
  );

  // 2. Get pull-source accounts this user follows
  const pullSources = await getPullSourcesForUser(userId);

  // 3. Fetch recent posts from pull sources
  const pulledPosts = await Promise.all(
    pullSources.map(authorId =>
      getRecentPosts(authorId, cursor, 5) // small limit per source
    )
  );

  // 4. Merge, rank, deduplicate, paginate
  const merged = mergeAndRank([...pushedPostIds, ...pulledPosts.flat()]);
  return paginate(merged, cursor, limit);
}
```

**Added complexity:**

- **Two code paths at read time:** Every feed read must check for pull sources and merge, even if there are none. Pure push has a single cache read path.
- **Maintaining the pull-source list:** You need a fast lookup of "which accounts that I follow are pull-source celebrities?" This is another data structure to maintain and cache.
- **Ranking consistency:** Posts from push and pull have different freshness guarantees. Pushed posts may be seconds old; pulled posts are fetched in real time. The ranking algorithm must handle this without penalizing either.
- **Threshold management:** The follower count changes over time. A user crossing the threshold requires migrating their fan-out strategy, which means their existing pushed posts remain in caches while new posts switch to pull.

Despite the complexity, hybrid is what every large-scale feed system uses because the alternative — either slow reads or catastrophic write amplification — is worse.

</details>

<details>
<summary>7. What is the celebrity problem (hot key / fan-out explosion) and why does it break a pure fan-out-on-write system — walk through the math of what happens when a user with 10 million followers posts, how does the hybrid solution solve this, and what other mitigations exist beyond simply switching to pull for high-follower accounts?</summary>

**The celebrity problem:**

In a pure push system, one post = one write per follower. When a celebrity with 10M followers posts:

- **10 million write operations** to individual feed caches
- At 10K writes/sec per worker, that's **~17 minutes** to complete fan-out with a single worker, or 100+ workers needed for sub-10-second delivery
- Those 10M messages flood the fan-out queue, **blocking fan-out for every other post** behind it
- The celebrity's follower list itself becomes a hot key in the social graph database
- The Redis cluster handling those 10M writes experiences a massive spike on specific shards

One celebrity posting causes cascading delays across the entire system. If multiple celebrities post around the same time (e.g., during a live event), the queue grows unboundedly.

**How hybrid solves it:**

As covered in question 6, the hybrid approach skips fan-out entirely for accounts above the follower threshold. The celebrity's post is stored once, and followers fetch it at read time. This converts 10M writes into at most 10M reads over time — but those reads are spread out (not all followers check their feed simultaneously) and served from a heavily cached post object.

**Other mitigations beyond the hybrid switch:**

1. **Tiered fan-out priority:** Fan out to active users first (logged in within the last N hours). Inactive users' caches can be populated lazily if they return. This reduces the effective fan-out count by 50-70% for most posts.

2. **Rate limiting the fan-out queue per author:** A celebrity's fan-out job is broken into batches (e.g., 10K followers per message). These batches are interleaved with other authors' fan-out jobs so no single post monopolizes the queue.

3. **Separate queues for high-follower accounts:** Route fan-out jobs for accounts above a certain threshold to a dedicated queue with its own worker pool, isolating the blast radius from normal traffic.

4. **Pre-computed follower segments:** Partition a celebrity's followers into segments stored in separate shards. Fan-out workers process segments in parallel across different machines, avoiding a single hot key for the follower list.

5. **Probabilistic fan-out:** For extremely high-follower accounts, push only to the most engaged followers (those who frequently interact with the celebrity's content) and use pull for the rest. This is a more granular version of hybrid.

6. **Coalescing:** If a celebrity posts 3 times in quick succession, batch all 3 posts into a single fan-out pass rather than 3 separate passes of 10M writes each.

</details>

## Component Deep Dives

<details>
<summary>8. How does feed ranking work — what signals (recency, engagement, affinity, content type) feed into a ranking model, how do you compute a relevance score, and what is the tradeoff between write-time placement (pre-ranked during fan-out) vs read-time ranking (scored on fetch)?</summary>

**Ranking signals:**

| Signal | Description | Example |
|---|---|---|
| **Recency** | How new the post is | Exponential decay: score drops over hours |
| **Engagement** | Likes, comments, shares, click-through | A post with 10K likes in 30 min ranks higher |
| **Affinity** | How much this user interacts with the author | Frequently liked authors get boosted |
| **Content type** | Image, video, text, link | Video might get a multiplier based on platform goals |
| **Social proof** | "Your friend liked this" | Mutual connections engaging signals relevance |
| **Negative signals** | "See fewer posts like this", muted keywords | Demotes or filters |

**Computing a relevance score:**

At its simplest, a weighted linear combination:

```
score = w1 * recency_score
      + w2 * engagement_score
      + w3 * affinity_score
      + w4 * content_type_boost
      + w5 * social_proof_score
```

At scale (Facebook, Twitter/X), this is a machine learning model — a lightweight neural network or gradient-boosted tree trained on engagement data. The model predicts P(user engages with this post) and uses that as the score.

For an interview, the linear model is sufficient. The key insight is that ranking transforms a chronological feed into a relevance-ordered feed.

**Write-time vs read-time ranking:**

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| **Write-time** | Compute score during fan-out, store as the sorted set score in Redis | Fast reads — already sorted | Score is frozen at write time. Engagement changes (likes accumulating) don't update the score. Requires re-scoring to stay fresh. |
| **Read-time** | Store posts with timestamp as score, re-rank the top N posts at read time | Always uses latest signals (current engagement, current affinity) | Adds latency to every read. Must fetch signal data for N posts per request. |
| **Hybrid** | Use timestamp as the base score at write time, apply a lightweight re-ranking pass at read time on the top 200-500 candidates | Balances freshness and latency | More complex but what production systems actually do |

**The practical approach:** Store posts in the feed cache sorted by timestamp. At read time, fetch ~200 candidates (more than the page size), score them with a lightweight ranker using fresh signals, return the top 20. This keeps write-time simple (just push post IDs) and concentrates ranking complexity at read time where it has access to current data.

</details>

<details>
<summary>9. How would you deliver real-time feed updates to connected clients — compare polling, long polling, SSE, and WebSockets for this use case, what are the cost and scaling implications of maintaining persistent connections at scale, and when is simple polling actually the better choice?</summary>

| Method | How it works | Latency | Server cost | Complexity |
|---|---|---|---|---|
| **Polling** | Client requests `/feed/new?since=timestamp` every N seconds | N/2 seconds avg | Low per connection, high aggregate (many empty responses) | Lowest |
| **Long polling** | Client requests, server holds connection until new data or timeout | Near real-time | Moderate (held connections consume threads/memory) | Low-moderate |
| **SSE** | Server pushes events over a single HTTP connection | Real-time | High (persistent connections) | Moderate |
| **WebSockets** | Full-duplex persistent connection | Real-time | Highest (stateful connections, harder to load balance) | Highest |

**Cost at scale:**

With 300M DAU and ~50M concurrent users, persistent connections mean 50M open sockets. Each connection consumes:
- Memory: ~10-50KB per connection = 500GB-2.5TB just for connection state
- File descriptors and OS resources
- Load balancer complexity: sticky sessions or connection-aware routing
- Deployment complexity: graceful connection draining during deploys

**When polling wins:**

- Feeds don't need sub-second updates — a 15-30 second delay is perfectly acceptable for most social feeds
- Polling at 30-second intervals for 50M concurrent users = ~1.7M requests/second, which is manageable and stateless
- Stateless infrastructure is simpler to scale, deploy, and debug
- Mobile clients with unreliable connectivity handle polling gracefully (just retry), while persistent connections require reconnection logic

**Practical recommendation for a news feed:**

Use **polling with adaptive intervals** as the baseline. Poll every 30 seconds when the app is in the foreground, stop when backgrounded. If a notification system exists, use push notifications for high-priority content and let the client refresh the feed on open.

For a premium real-time experience (e.g., live events, Twitter-like breaking news), add **SSE** for users who are actively viewing the feed. SSE is simpler than WebSockets (unidirectional, works over HTTP/2, auto-reconnects) and sufficient since the client doesn't need to send data back over this channel.

WebSockets are overkill for feeds unless you also need real-time typing indicators, live comments, or other bidirectional features.

</details>

<details>
<summary>10. Walk through the internals of the fan-out service — how does it look up a poster's followers, batch the writes to individual feed caches, distribute work across multiple workers, and handle partial failures where some followers' feeds are updated but others fail mid-batch?</summary>

**Step-by-step flow:**

**1. Event consumption and job creation:**

The fan-out service consumes `PostCreated` events from a Kafka topic (or SQS queue). Each event contains `{ postId, authorId, timestamp }`. The consumer checks the author's follower count — if above the hybrid threshold, it drops the event (pull-based delivery). Otherwise, it creates a fan-out job.

**2. Follower lookup:**

The service queries the social graph service for the author's follower list. For large follower lists, this is a paginated/streamed response, not a single query:

```typescript
async function* getFollowerBatches(authorId: string, batchSize = 5000) {
  let cursor: string | null = null;
  do {
    const { followers, nextCursor } = await socialGraphService.getFollowers(
      authorId, { cursor, limit: batchSize }
    );
    yield followers;
    cursor = nextCursor;
  } while (cursor);
}
```

**3. Batching and distribution:**

Each batch of follower IDs becomes a sub-task message published to a fan-out work queue. Workers pull these sub-tasks independently, enabling parallel processing:

```typescript
for await (const followerBatch of getFollowerBatches(authorId)) {
  await fanoutQueue.publish({
    postId,
    authorId,
    timestamp,
    followerIds: followerBatch, // 5000 IDs per message
  });
}
```

**4. Worker execution — batched Redis writes:**

Each worker receives a batch and uses Redis pipelining to write efficiently:

```typescript
async function processFanoutBatch(job: FanoutJob) {
  const pipeline = redis.pipeline();

  for (const followerId of job.followerIds) {
    const feedKey = `feed:${followerId}`;
    pipeline.zadd(feedKey, job.timestamp, job.postId);
    pipeline.zremrangebyrank(feedKey, 0, -(MAX_FEED_SIZE + 1)); // trim to limit
  }

  await pipeline.exec(); // single round-trip for the entire batch
}
```

Pipelining turns 5000 individual Redis commands into a single network round-trip, dramatically reducing latency.

**5. Partial failure handling:**

Redis pipeline responses return per-command results. If some writes succeed and others fail:

- **Retry the failed subset:** Extract the follower IDs whose writes failed and republish a smaller retry batch to the queue. Use exponential backoff with a retry limit (e.g., 3 attempts).
- **Dead letter queue:** After max retries, move failed follower IDs to a DLQ for manual inspection or background reconciliation.
- **Idempotent writes:** `ZADD` is idempotent — writing the same `(score, postId)` twice has no effect. This makes retries safe without deduplication logic.

```typescript
const results = await pipeline.exec();
// i * 2 because each follower has two pipeline commands: ZADD + ZREMRANGEBYRANK
const failedFollowers = job.followerIds.filter((_, i) => results[i * 2][0] !== null);

if (failedFollowers.length > 0 && job.retryCount < MAX_RETRIES) {
  await fanoutQueue.publish({
    ...job,
    followerIds: failedFollowers,
    retryCount: job.retryCount + 1,
  });
}
```

**Worker scaling:** Workers are stateless and horizontally scalable. Auto-scaling is driven by queue depth — when pending messages exceed a threshold, spin up more workers. During quiet periods, scale down to a baseline.

</details>

<details>
<summary>11. How would you design the feed cache layer — why use Redis, what data structure (sorted set vs list) works best and why, should you store full post objects or just post IDs in the cache, how do you size the cache (how many posts per user), and what eviction strategy handles the long tail of inactive users?</summary>

**Why Redis:**

- Sub-millisecond read latency for feed fetches
- Native sorted set operations for ordered, paginated data
- Pipelining for efficient bulk fan-out writes
- Cluster mode for horizontal sharding
- Persistence options (RDB/AOF) for cache durability without relying solely on memory

**Sorted set vs list:**

| Aspect | Sorted Set (ZSET) | List |
|---|---|---|
| Ordering | By score (timestamp) — always sorted | Insertion order only |
| Range queries | `ZRANGEBYSCORE` for cursor-based pagination | `LRANGE` by index only |
| Deduplication | Automatic (same member can't exist twice) | No dedup — can have duplicate post IDs |
| Trimming | `ZREMRANGEBYRANK` to cap size | `LTRIM` works but no score-based trim |
| Insert position | Any position (sorted by score) | Head or tail only |

**Sorted set wins** because feeds need score-based ordering (for ranking), cursor-based pagination (range queries by score), and idempotent writes (no duplicates from retried fan-out).

**Post IDs vs full objects:**

Store **only post IDs** (and the score) in the feed sorted set. Reasons:

1. **Space efficiency:** A post ID is ~36 bytes (UUID). A full post object is 1-5KB. With 500 entries per user and 300M users, that's the difference between ~5TB and ~250TB+.
2. **Update propagation:** When a post is edited, you update it in one place (the post store). If full objects were in every follower's feed cache, you'd need to update millions of copies.
3. **Flexibility:** The same post object cache is shared across feeds, user profiles, permalink pages — single cache, multiple consumers.

The tradeoff is an extra lookup to hydrate IDs into objects at read time, but post objects are themselves cached (see question 12).

**Sizing:**

Store the **most recent 500 posts** per user's feed. This covers ~2-3 days of scrolling for an active user. Beyond 500, the user hits "end of cached feed" and you either reconstruct from the database or show a "you're all caught up" message.

- 500 entries x ~50 bytes per entry (ID + score) = ~25KB per user
- 300M users x 25KB = ~7.5TB total feed cache

In practice, many users are inactive and their caches expire, so actual usage is lower.

**Eviction strategy:**

- **Per-user trimming:** After each `ZADD`, run `ZREMRANGEBYRANK` to keep only the top 500 entries. This caps per-user size.
- **TTL for inactive users:** Set a TTL (e.g., 7 days) on each feed key. If a user doesn't log in, their feed cache expires. When they return, rebuild from the database (question 13). This reclaims the long tail — most of the 300M users aren't daily active.
- **Touch on read:** Reset the TTL every time the feed is read, keeping active users warm.

</details>

<details>
<summary>12. Walk through the feed read path for a normal user — starting from a cache lookup, how do you hydrate post IDs into full post objects, merge in celebrity posts that were pulled at read time rather than pushed, and return a ranked, filtered page of results using cursor-based pagination?</summary>

**Complete read path:**

```
Client → API Gateway → Feed Service → Redis (feed cache)
                                     → Post Cache (hydration)
                                     → Celebrity Post Fetch (pull merge)
                                     → Rank & Filter → Response
```

**Step 1 — Cache lookup:**

```typescript
// Decode cursor to get the score boundary
const maxScore = cursor ? decodeCursor(cursor).score : '+inf';

// Fetch more than needed to account for filtering and merge
const candidateCount = limit * 3; // e.g., 60 candidates for 20 results
// ZREVRANGEBYSCORE is deprecated since Redis 6.2.0
// Modern equivalent: ZRANGE feed:${userId} ${maxScore} -inf BYSCORE REV WITHSCORES LIMIT 0 ${candidateCount}
const postEntries = await redis.zrange(
  `feed:${userId}`, maxScore, '-inf',
  'BYSCORE', 'REV', 'WITHSCORES', 'LIMIT', 0, candidateCount
);
// Returns: [postId1, score1, postId2, score2, ...]
```

**Step 2 — Celebrity post merge:**

```typescript
// Get list of pull-source (celebrity) accounts this user follows
const pullSources = await getPullSourcesForUser(userId);

if (pullSources.length > 0) {
  // Fetch recent posts from each celebrity, bounded by same cursor
  const celebrityPosts = await Promise.all(
    pullSources.map(authorId =>
      postStore.getRecentByAuthor(authorId, { before: maxScore, limit: 5 })
    )
  );

  // Merge into candidate set
  postEntries.push(...celebrityPosts.flat());
}
```

**Step 3 — Hydration:**

Convert post IDs into full post objects. Use a multi-get from the post cache (Redis or Memcached) to batch this:

```typescript
const postIds = postEntries.map(e => e.postId);
const posts = await postCache.mget(postIds); // single round-trip

// Cache misses: fetch from database, backfill cache
const missingIds = postIds.filter((id, i) => !posts[i]);
if (missingIds.length > 0) {
  const dbPosts = await postStore.getByIds(missingIds);
  await postCache.mset(dbPosts); // backfill
  // merge into posts array
}
```

**Step 4 — Filter:**

Remove posts the user shouldn't see:
- Deleted posts (`is_deleted = true`)
- Posts from blocked/muted users
- Posts the user has already marked "not interested"
- Duplicate posts (same post from reshares)

**Step 5 — Rank:**

Apply the lightweight re-ranking pass (as discussed in question 8) on the filtered candidate set using fresh signals.

**Step 6 — Paginate and respond:**

```typescript
const ranked = rankPosts(filtered, userId);
const page = ranked.slice(0, limit);
const nextCursor = page.length === limit
  ? encodeCursor({ score: page[page.length - 1].score, postId: page[page.length - 1].postId })
  : null;

return { posts: page, nextCursor };
```

The entire path targets <200ms: ~1ms for Redis sorted set read, ~2-5ms for post cache multi-get, ~5-10ms for celebrity merge, ~5ms for ranking. The slowest part is typically database fallback on cache misses.

</details>

<details>
<summary>13. What happens when a user's feed cache is completely empty (cache miss) — how do you reconstruct their feed on the fly, what is the performance cost compared to a cache hit, and what strategies (pre-warming, lazy population, fallback to a global trending feed) prevent cold cache reads from degrading the user experience?</summary>

**Why a cache miss happens:**
- User hasn't logged in for a while and their feed TTL expired
- Cache node failure or eviction under memory pressure
- New user who just started following accounts

**Reconstruction on the fly:**

```typescript
async function reconstructFeed(userId: string): Promise<void> {
  // 1. Get accounts this user follows
  const following = await socialGraphService.getFollowing(userId);

  // 2. Fetch recent posts from each followed account
  const recentPosts = await Promise.all(
    following.map(authorId =>
      postStore.getRecentByAuthor(authorId, { limit: 10 })
    )
  );

  // 3. Merge-sort by timestamp, take top 500
  const merged = mergeSort(recentPosts.flat(), (a, b) => b.createdAt - a.createdAt)
    .slice(0, 500);

  // 4. Write back to cache
  const pipeline = redis.pipeline();
  for (const post of merged) {
    pipeline.zadd(`feed:${userId}`, post.createdAt, post.postId);
  }
  pipeline.expire(`feed:${userId}`, FEED_TTL);
  await pipeline.exec();
}
```

**Performance cost:**

| Scenario | Latency | DB Load |
|---|---|---|
| Cache hit | ~5-10ms | None |
| Cache miss (reconstruct) | 200-2000ms | High: N queries (one per followed account) |

If a user follows 200 accounts and each query takes 5ms, that's 1 second of parallel database queries plus merge time. This is 100x slower than a cache hit.

**Mitigation strategies:**

**1. Lazy population with immediate fallback:**
Don't block the user while reconstructing. Return a fallback feed immediately, trigger async reconstruction:

```typescript
async function getFeed(userId: string, cursor: string, limit: number) {
  const cached = await redis.zrevrangebyscore(`feed:${userId}`, ...);

  if (cached.length === 0) {
    // Trigger async reconstruction
    reconstructionQueue.publish({ userId });

    // Return fallback immediately
    return getFallbackFeed(userId, limit);
  }

  return buildFeedResponse(cached, cursor, limit);
}
```

**2. Fallback to trending/global feed:**
Maintain a pre-computed "trending" or "popular" feed that's always available. Show this with a "Catching up on your feed..." message. Mix in posts from the user's followed accounts as they're fetched.

**3. Pre-warming on login signal:**
When a user's session starts (login, app open), proactively check if their feed cache exists. If not, trigger reconstruction before they navigate to the feed tab. Mobile apps often have a splash screen or home page that buys time for this.

**4. Request coalescing:**
If a user's feed cache is cold and they refresh 5 times impatiently, don't trigger 5 reconstructions. Use a lock or semaphore so only one reconstruction runs per user, and subsequent requests wait for it:

```typescript
const lockKey = `feed-rebuild:${userId}`;
const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 30);
if (!acquired) {
  // Another reconstruction is in progress — wait or return fallback
  return getFallbackFeed(userId, limit);
}
```

**5. Partial reconstruction:**
Instead of fetching from all 200 followed accounts, reconstruct from the top 20-30 most interacted-with accounts first. This gives a "good enough" feed quickly, and a background job fills in the rest.

</details>

<details>
<summary>14. How do you handle post updates and deletes in a fan-out-on-write system — when a user edits or deletes a post that has already been fanned out to millions of feeds, how do you propagate that change without re-fanning, what consistency guarantees can you realistically offer, why does storing only post IDs in the feed cache simplify this problem, and what tombstone or soft-delete mechanism prevents deleted posts from reappearing during cache rebuilds?</summary>

**Why post IDs make this tractable:**

As covered in question 11, feed caches store only `(postId, score)` pairs. The actual post content lives in a single canonical location (post database + post cache). This means:

- **Edit:** Update the post in the database and invalidate/update the post cache. The next time any feed hydrates that post ID, it gets the updated version. Zero fan-out work.
- **Delete:** Mark the post as deleted in the database and remove it from the post cache. During hydration, deleted posts are filtered out.

If feed caches stored full post objects, you'd need to update millions of copies across all followers' caches — essentially re-fanning the edit.

**Delete flow:**

```typescript
async function deletePost(postId: string, authorId: string) {
  // 1. Soft-delete in database
  await db.query('UPDATE posts SET is_deleted = true, deleted_at = NOW() WHERE post_id = $1', [postId]);

  // 2. Remove from post cache
  await postCache.del(`post:${postId}`);

  // 3. Optionally: publish PostDeleted event for eager cleanup
  await eventBus.publish('PostDeleted', { postId, authorId });
}
```

**Eager vs lazy cleanup of feed caches:**

- **Lazy (recommended for most cases):** Don't touch feed caches at all. When a feed is read and the post ID is hydrated, the hydration step discovers `is_deleted = true` and filters it out. The stale entry remains in the sorted set until it's either trimmed by size limit or overwritten.
- **Eager:** Publish a `PostDeleted` event, fan out the delete to all followers' caches, and `ZREM` the post ID. This is the same fan-out cost as the original write, which defeats the purpose of storing IDs only. Use this only for legal/compliance takedowns where you need to guarantee the content is gone from all caches within minutes.

**Consistency guarantees:**

Realistically, **eventual consistency with a short window**. After a delete:
- Users who already have the hydrated post in their client-side view may see it until they refresh
- Users whose feed cache still contains the post ID will not see it on their next fetch (filtered during hydration)
- The window is effectively the post cache TTL — usually seconds

For edits, the window is even smaller since edits just change content, not visibility.

**Tombstone mechanism for cache rebuilds:**

During feed reconstruction (question 13), you query recent posts from followed accounts. Without protection, a hard-deleted post (physically removed from the database) could have its ID still referenced in some feed cache that hasn't been trimmed. When you try to hydrate it, you get a cache miss, query the database, and get nothing back.

Soft deletes solve this: keep the row with `is_deleted = true` for a retention period (e.g., 30 days), then hard-delete. During hydration, soft-deleted posts are explicitly filtered out. The retention period must exceed the maximum feed cache TTL to ensure no feed cache references a hard-deleted post.

```sql
-- Periodic cleanup: hard-delete posts soft-deleted more than 30 days ago
DELETE FROM posts WHERE is_deleted = true AND deleted_at < NOW() - INTERVAL '30 days';
```

</details>

## Scaling & Failure Handling

<details>
<summary>15. How do you scale the fan-out system when a viral post hits — what happens to worker load and queue depth when a high-follower account posts content that also gets reshared widely, how do you implement backpressure to prevent the fan-out queue from overwhelming downstream services, and why should you prioritize active users over inactive ones during fan-out?</summary>

**What happens during a viral post:**

A celebrity with 5M followers posts. Simultaneously, thousands of users reshare it, each triggering their own fan-out. The cascade:

1. The original post triggers fan-out to 5M followers (if below the hybrid threshold) or gets batched into 1000 sub-tasks of 5K followers each
2. Each reshare triggers fan-out to that resharer's followers — potentially millions more writes
3. Queue depth spikes from a baseline of ~10K pending messages to millions
4. Workers saturate, Redis write throughput spikes, latency increases for all fan-out jobs
5. Fan-out lag increases for everyone — a normal user's post that would normally reach followers in 2 seconds now takes minutes

**Backpressure mechanisms:**

**1. Queue depth-based auto-scaling:**
```
if queue_depth > 100K → scale workers to 2x baseline
if queue_depth > 500K → scale workers to 5x baseline
if queue_depth > 1M  → activate emergency mode
```

**2. Rate limiting per-author fan-out:**
Cap the rate at which any single author's fan-out jobs are processed. Interleave batches from different authors so one celebrity doesn't monopolize the worker pool:

```typescript
// Workers pull from multiple priority queues round-robin
const job = await Promise.race([
  highPriorityQueue.pull(),  // normal users (fast fan-out)
  lowPriorityQueue.pull(),   // celebrities, reshare cascades
]);
```

**3. Downstream circuit breaking:**
If Redis write latency exceeds a threshold (e.g., >50ms per pipeline), workers back off and reduce batch sizes. This prevents overwhelming Redis and causing cascading failures:

```typescript
if (redisLatencyP99 > 50) {
  await sleep(exponentialBackoff(retryCount));
  reduceBatchSize();
}
```

**4. Drop or defer for inactive users:**
If queue depth exceeds a critical threshold, stop fan-out to inactive users entirely. Their feed caches will be rebuilt when they return (question 13).

**Why prioritize active users:**

- Active users (logged in recently) will actually read their feed — fan-out to them produces visible value
- Inactive users (haven't logged in for days/weeks) won't see the post for hours or days anyway. Their feed cache may expire before they return, wasting the write entirely
- During a viral event, 50-70% of followers may be inactive. Skipping them cuts fan-out volume in half immediately

**Implementation:** Maintain an "active users" bitmap or set (updated on login/session activity). During fan-out, check each follower batch against this set and route inactive followers to a lower-priority queue or skip them entirely.

**Emergency mode for truly viral events:**
When queue depth grows unboundedly despite scaling, dynamically raise the hybrid threshold — temporarily switch more authors to pull-based delivery. This is a knob you can turn in real time via a configuration service.

</details>

<details>
<summary>16. How would you partition and shard the news feed system — why shard the feed cache by user ID and post storage by post ID, what problems does each sharding key solve, and how do you handle hot shards (e.g., a celebrity's post storage shard or a viral post being read from millions of feed caches on the same shard)?</summary>

**Sharding the feed cache by user ID:**

Each user's feed is a self-contained sorted set. Sharding by `hash(userId) % N` ensures:

- A user's entire feed lives on one shard — no cross-shard reads for a single feed fetch
- Fan-out writes are distributed across shards (followers are spread across the user space)
- Load is proportional to user activity, which is roughly uniform across shards

```
Shard 0: feed:userA, feed:userD, feed:userG ...
Shard 1: feed:userB, feed:userE, feed:userH ...
Shard 2: feed:userC, feed:userF, feed:userI ...
```

Redis Cluster handles this natively with hash slots — `HASH_SLOT(feed:{userId})` routes to the correct node.

**Sharding post storage by post ID:**

Posts need to be accessed individually (hydration) and are written once. Sharding by `hash(postId) % N` ensures:

- Write distribution is uniform (post IDs are UUIDs, inherently well-distributed)
- Individual post lookups hit a single shard
- No hot shard from a single author's posts (unlike sharding by authorId, which would concentrate all of a celebrity's posts on one shard)

**Why not shard posts by author ID:**

If you shard by author ID, a celebrity's shard stores all their posts and handles all hydration requests for those posts. When a viral post is fetched by millions of users, that single shard gets hammered.

**Hot shard scenarios and mitigations:**

**1. Viral post being hydrated from one shard:**

Even with post ID sharding, a single viral post lives on one shard and gets millions of reads. Solutions:

- **Post object cache with replication:** Cache the post object in a distributed cache (Redis/Memcached) with multiple replicas. The post cache is separate from the feed cache and tuned for read-heavy access with short TTLs.
- **Local in-memory cache on feed service nodes:** Cache the top N hottest posts in process memory (LRU with 60-second TTL). This absorbs the majority of reads without hitting Redis at all.

```typescript
// Two-tier caching for hot posts
const post = localCache.get(postId)        // L1: in-memory, ~100μs
  ?? await postCache.get(postId)            // L2: Redis, ~1ms
  ?? await postStore.getById(postId);       // L3: database, ~10ms
```

**2. Celebrity's feed cache shard:**

Celebrities themselves don't have abnormally large feeds (they follow a normal number of accounts). The hot shard issue is about reading a celebrity's *posts*, not their *feed*. This is handled by the post-level caching above.

**3. Feed cache shard receiving disproportionate fan-out writes:**

During fan-out, writes hit whichever shards host the followers' feed caches. If a celebrity's followers happen to cluster on certain shards (unlikely with hash-based sharding but possible with range-based), those shards spike. Mitigation: ensure hash-based sharding for uniform distribution, and use Redis Cluster's built-in rebalancing.

**Consistent hashing for shard additions:**

When adding or removing shards, consistent hashing minimizes key remapping. Redis Cluster handles this via hash slot migration. For the post database, use consistent hashing with virtual nodes so adding a shard moves ~1/N of the data rather than reshuffling everything.

</details>

<details>
<summary>17. What happens when the feed cache layer fails — how do you prevent a database stampede when millions of users suddenly hit the post database directly, what role do request coalescing and circuit breaking play, and how do you serve a degraded but functional feed while the cache recovers?</summary>

**The stampede scenario:**

Redis cluster goes down (network partition, OOM, or node failure). Millions of concurrent feed requests get cache misses simultaneously and all fall through to the post database. The database, sized for ~5% cache miss rate, gets hit with 100x its normal load and collapses. This is a classic cache stampede (thundering herd).

**Prevention layer 1 — Request coalescing (singleflight):**

When multiple requests for the same user's feed arrive simultaneously, only one actually hits the database. Others wait for that result:

```typescript
import { Singleflight } from './singleflight';

const singleflight = new Singleflight();

async function getFeedWithCoalescing(userId: string) {
  return singleflight.do(`feed:${userId}`, async () => {
    // Only one execution per key, all waiters get the same result
    return reconstructFeedFromDB(userId);
  });
}
```

This collapses N concurrent requests for the same feed into 1 database query.

**Prevention layer 2 — Circuit breaking:**

If the database error rate or latency exceeds thresholds, stop sending traffic to it entirely:

```typescript
const circuitBreaker = new CircuitBreaker({
  failureThreshold: 50,    // 50% error rate
  resetTimeout: 30_000,    // try again after 30s
  halfOpenRequests: 5,     // test with 5 requests before fully opening
});

async function getFeed(userId: string) {
  return circuitBreaker.execute(async () => {
    const cached = await redis.zrevrangebyscore(`feed:${userId}`, ...);
    if (cached.length > 0) return hydrate(cached);
    return reconstructFeedFromDB(userId);
  });
}
```

When the circuit is open, requests immediately get the fallback response without touching the database.

**Prevention layer 3 — Rate limiting database queries:**

Cap the number of concurrent feed reconstruction queries hitting the database:

```typescript
const dbSemaphore = new Semaphore(100); // max 100 concurrent reconstructions

async function reconstructFeedFromDB(userId: string) {
  const permit = await dbSemaphore.acquire(/* timeout */ 5000);
  if (!permit) throw new Error('Reconstruction rate limited');
  try {
    return await doReconstruction(userId);
  } finally {
    permit.release();
  }
}
```

**Serving a degraded feed:**

When the cache is down and the database is protected by circuit breakers, you still need to return something:

1. **Global trending feed:** A pre-computed list of popular posts, stored redundantly (separate Redis instance, CDN, or even hardcoded in the application). Always available.
2. **Client-side cache:** The mobile/web client likely has the last fetched feed in local storage. Return a 503 with a `Retry-After` header, and the client shows stale content.
3. **Stale cache reads:** If using Redis Cluster and only some nodes are down, read from replicas even if they're slightly stale. Configure `READONLY` mode on replicas.
4. **Partial feed:** If you can reach the database at a reduced rate, serve a simplified feed (fewer posts, no ranking, no celebrity merge) to reduce the query cost per request.

**Recovery sequence:**

1. Cache nodes come back online (empty)
2. Circuit breaker enters half-open state, allows limited traffic
3. Feeds are reconstructed lazily as users request them (with coalescing and rate limiting)
4. Active user feeds warm up over 10-30 minutes
5. Circuit breaker fully closes as error rates drop

The key principle: every layer degrades gracefully rather than failing catastrophically. Users get a worse experience (stale/trending feed) rather than no experience (error page).

</details>

<details>
<summary>18. A user reports that new posts from accounts they follow are not appearing in their feed for several minutes — walk through how you would diagnose this end-to-end: checking the fan-out queue depth, verifying the social graph has the follow relationship, inspecting the user's feed cache, and determining whether the issue is fan-out lag, a cache miss, or a ranking/filtering bug that's suppressing the content.</summary>

**Systematic diagnosis — work backwards from the symptom:**

**Step 1 — Verify the symptom is real:**

Check the specific post the user expects to see. Confirm it exists in the post database, is not deleted, and was created at the time the user claims. Get the `postId` and `authorId`.

```sql
SELECT post_id, author_id, is_deleted, created_at FROM posts WHERE post_id = '<postId>';
```

If the post is soft-deleted or marked as spam, that's your answer.

**Step 2 — Verify the follow relationship:**

Confirm the user actually follows the author in the social graph:

```sql
SELECT * FROM follows WHERE follower_id = '<userId>' AND followee_id = '<authorId>';
```

If the row is missing, the user unfollowed (intentionally or due to a bug) and the fan-out correctly skipped them. Check the follow audit log for recent unfollow events.

**Step 3 — Check the author's fan-out path:**

Is the author above the hybrid threshold? If yes, their posts aren't pushed — they're pulled at read time. Check whether the pull-merge logic is functioning:

```typescript
// Is this author a pull source?
const followerCount = await getFollowerCount(authorId);
// Compare against threshold
```

If it's a pull source, the issue is in the read-time merge (Step 6). If it's a push source, continue to Step 4.

**Step 4 — Check fan-out queue depth and lag:**

Look at the fan-out queue metrics:
- Current queue depth (is there a backlog?)
- Consumer lag per partition (Kafka) or approximate message age (SQS)
- Fan-out latency p50/p99 for the last hour

If queue depth is abnormally high (e.g., a viral post is clogging the queue), the post's fan-out job may be waiting behind millions of other jobs. Check the timestamp of the oldest unprocessed message.

```
Fan-out queue depth: 2.3M messages (normal: ~10K)
Oldest message age: 8 minutes
→ Diagnosis: fan-out lag due to queue congestion
```

**Step 5 — Inspect the user's feed cache:**

Check if the post ID exists in the user's Redis sorted set:

```bash
ZSCORE feed:<userId> <postId>
```

- **Post ID present:** Fan-out succeeded. The issue is downstream (hydration, filtering, or ranking). Go to Step 6.
- **Post ID absent:** Fan-out hasn't reached this user yet (lag) or failed. Check the DLQ for failed fan-out jobs mentioning this user.
- **Feed key doesn't exist:** The user's entire feed cache has expired. The issue is cache miss reconstruction (question 13).

**Step 6 — Check hydration and filtering:**

If the post ID is in the cache but not appearing in the feed response, it's being filtered or ranked out:

- Is the post failing hydration? (Post cache miss + database lookup returning deleted?)
- Is the author on the user's mute or block list?
- Is the ranking algorithm scoring it so low it falls below the page cutoff?
- Is there a content filter (spam, sensitive content) suppressing it?

Query the ranking service with debug mode to see the post's score and why it was ranked where it was.

**Common root causes by frequency:**

1. **Fan-out lag** (most common during load spikes) — queue depth too high
2. **Hybrid pull-merge bug** — celebrity posts not being merged at read time
3. **Cache miss** — user's feed expired and reconstruction didn't include the post
4. **Ranking suppression** — post scored too low due to stale affinity signals
5. **Follow graph inconsistency** — follow event not propagated to the social graph service

</details>

<details>
<summary>19. Your fan-out queue depth is growing unboundedly and workers can't keep up after a celebrity with 50 million followers posts — walk through how you handle this in real time: what circuit breakers activate, how do you switch that post to pull-based delivery, how do you prioritize which followers get the update first, and what metrics would you monitor to detect this situation before it cascades?</summary>

Building on the scaling mechanisms from question 15 (worker auto-scaling by queue depth, priority queues for active vs inactive users, dynamic hybrid threshold adjustment), this answer focuses on the real-time detection, automated incident response, and mid-fan-out cancellation specific to a 50M-follower scenario.

**Detection — metrics that fire alerts:**

| Metric | Normal | Alert threshold |
|---|---|---|
| Fan-out queue depth | <50K | >500K |
| Queue depth growth rate | Stable | >100K/min sustained |
| Fan-out latency p99 | <5s | >30s |
| Worker CPU utilization | 40-60% | >85% |
| Redis write latency p99 | <5ms | >20ms |

The combination of queue depth >500K AND growth rate >100K/min triggers the automated response. A single metric spike might be transient, but both together indicate the system can't keep up.

**Automated response playbook:**

**Phase 1 — Scale workers (0-2 minutes):**

Auto-scaling kicks in per the queue depth thresholds described in question 15. If scaling alone stabilizes queue depth, no further action needed.

**Phase 2 — Mid-fan-out cancellation and switch to pull (2-5 minutes):**

If queue depth continues growing despite scaling, the circuit breaker activates and cancels remaining fan-out for the offending post mid-flight:

```typescript
async function onFanoutJobReceived(job: FanoutJob) {
  const queueDepth = await getQueueDepth();

  if (queueDepth > CRITICAL_THRESHOLD) {
    const followerCount = await getFollowerCount(job.authorId);

    if (followerCount > EMERGENCY_THRESHOLD) {
      // Cancel remaining fan-out batches for this post
      await cancelPendingBatches(job.postId);

      // Mark post for pull-based delivery going forward
      await markAsPullBased(job.postId, job.authorId);

      // Followers who already received the push keep it;
      // the rest get it via pull at read time (question 12)
      return;
    }
  }

  await processFanoutBatch(job);
}
```

This partial fan-out is safe because the feed read path already merges push and pull content. Some followers (say 5M out of 50M) got the push; the remaining 45M get it via pull seamlessly.

**Phase 3 — Active user priority and inactive user shedding (concurrent with Phase 1-2):**

As described in question 15, fan-out jobs are split by follower activity into high-priority and low-priority queues. During a crisis, low-priority (inactive user) jobs are dropped entirely.

**Phase 4 — Dynamic hybrid threshold adjustment:**

If multiple celebrities post during a live event and the situation persists, temporarily lower the hybrid threshold (as described in question 15) to shift more accounts to pull-based delivery. Revert after queue depth normalizes.

**Post-incident analysis and prevention:**

- Review whether the permanent hybrid threshold should be lowered based on the incident data
- Implement per-author fan-out rate caps so no single author consumes more than X% of queue capacity
- Add pre-emptive pull-based routing for "near-celebrity" accounts during known high-traffic windows (sports events, elections, product launches)
- Pre-warm standby worker pools before predictable traffic spikes to reduce scaling lag

</details>
