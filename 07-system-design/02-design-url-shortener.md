# System Design: URL Shortener

> **20 questions**

- Requirements gathering and back-of-the-envelope estimation: read/write ratio, requests per second, storage growth, bandwidth, cache memory sizing
- API design: endpoints for creation, redirection, custom aliases, expiration
- Keyspace calculation and short URL length
- Encoding strategies: base62 conversion, MD5/SHA256 hash truncation, pre-generated key service (KGS)
- Database selection: relational vs NoSQL for simple key-value access patterns
- Schema design: URL mapping table, indexes, TTL-based expiration and cleanup
- Redirect flow: 301 vs 302 tradeoffs for caching and analytics
- Caching layer: eviction policies (LRU, LFU, TTL), cache sizing for power-law traffic
- Analytics: async click tracking (geographic, referrer, device) without redirect latency
- Write path: race conditions, uniqueness checks, ID predictability mitigation
- Scaling: database sharding (key-based vs hash-based partitioning), distributed cache, stateless application servers, cache warming strategies

---

## Requirements & Estimation

<details>
<summary>1. How do you gather and prioritize functional and non-functional requirements for a URL shortener — what questions would you ask the interviewer to scope the problem, why is the read-to-write ratio the single most important characteristic to establish early, and how does it shape every subsequent design decision from caching to database choice?</summary>

**Questions to ask the interviewer (in priority order):**

1. **Scale**: How many new URLs per day/month? How many redirects per day? This establishes the read-to-write ratio.
2. **Features**: Do we need custom aliases? URL expiration/TTL? Analytics (click counts, geo, referrer)?
3. **URL length**: Should short URLs be as short as possible, or is a fixed length acceptable?
4. **Durability**: How long do URLs need to live? Forever? Configurable TTL?
5. **Users**: Is this an open API or authenticated? Do users manage their URLs (edit, delete)?
6. **Availability vs consistency**: Is it acceptable for a newly created URL to take a second before it's resolvable? (Usually yes.)
7. **Abuse prevention**: Do we need rate limiting, spam/phishing detection?

**Why read-to-write ratio is the most important signal:**

A typical URL shortener has a 100:1 or even 1000:1 read-to-write ratio — URLs are created once and clicked many times. This single number drives nearly every design choice:

- **Caching**: A 100:1 ratio means the read path dominates. Investing heavily in caching (even caching 20% of URLs covers the vast majority of reads due to power-law distribution) yields massive returns.
- **Database choice**: With reads dominating, you optimize the DB for read throughput. Read replicas, eventually-consistent reads, and denormalized schemas all make sense. If the ratio were closer to 1:1, you'd worry more about write throughput and consistency.
- **Scaling strategy**: You scale the read path independently of the write path. Stateless app servers behind a load balancer handle redirects; the write path (URL creation) can be a smaller, separately scaled service.
- **Redirect type**: The ratio influences whether 301 (permanent, browser-cached) or 302 (temporary, always hits your server) makes sense — 301 reduces server load but sacrifices analytics.
- **Replication**: High read ratio justifies read replicas at the database level, since replication lag is acceptable for reads but not for the write-then-immediately-read case (which is rare).


</details>

<details>
<summary>2. Walk through the back-of-the-envelope estimation for a URL shortener handling 100M new URLs per month — how do you derive reads per second from the read-to-write ratio, estimate storage growth over 5 years (including metadata), calculate cache memory requirements assuming power-law traffic distribution, and what bandwidth numbers should you sanity-check?</summary>

**Write throughput:**

- 100M new URLs/month = ~3.3M/day = ~40 URLs/second (writes)

**Read throughput (assuming 100:1 read-to-write ratio):**

- 40 writes/sec × 100 = ~4,000 redirects/second
- Peak traffic: 2-3x average = ~10,000 redirects/sec

**Storage over 5 years:**

Each URL record includes:
| Field | Size |
|-------|------|
| Short key (7 chars) | 7 bytes |
| Long URL (avg) | ~500 bytes |
| Created timestamp | 8 bytes |
| Expiration timestamp | 8 bytes |
| User ID | 16 bytes |
| Metadata overhead | ~60 bytes |
| **Total per record** | **~600 bytes** |

- 100M/month × 12 months × 5 years = 6 billion URLs
- 6B × 600 bytes = ~3.6 TB of raw data
- With indexes and overhead: ~5-7 TB total

**Cache sizing:**

Power-law distribution means ~20% of URLs generate ~80% of traffic. Cache the hottest 20%:

- Daily active URLs generating reads: if 4,000 reads/sec, many are repeat hits on popular URLs. Caching the top 20% of daily active URLs is sufficient.
- Conservative: cache 20% of all URLs created in the last month = 20M URLs × 600 bytes = ~12 GB
- This fits comfortably in a single Redis/Memcached instance, or a small cluster for redundancy.

**Bandwidth:**

- Read bandwidth: 4,000 req/sec × ~1 KB (request + response headers) = ~4 MB/sec = ~330 GB/day
- Write bandwidth: 40 req/sec × ~1 KB = ~40 KB/sec (negligible)

**Sanity checks to call out:**

- Storage grows linearly and predictably — no surprises
- Cache hit rate should be 90%+ given power-law; if it's lower, your cache is undersized or your eviction policy is wrong
- The redirect response is tiny (just headers + 302 + Location), so bandwidth is rarely the bottleneck — CPU and DB I/O are

</details>

<details>
<summary>3. How do you calculate the required short URL length — given a target keyspace (e.g., 100M URLs/month for 10 years), why does base62 encoding determine the minimum character count, what is the tradeoff between shorter URLs (better UX) and larger keyspace (collision safety), and at what point do you need 7 vs 8 characters?</summary>

**Keyspace requirement:**

100M URLs/month × 12 months × 10 years = 12 billion unique URLs needed.

**Base62 encoding:**

Base62 uses `[a-zA-Z0-9]` — 62 characters. The number of unique strings of length N is 62^N:

| Length | Unique keys | Enough? |
|--------|------------|---------|
| 6 | 62^6 = ~56.8 billion | Yes, but thin margin |
| 7 | 62^7 = ~3.5 trillion | Yes, with massive headroom |
| 8 | 62^8 = ~218 trillion | Overkill for this scale |

**7 characters gives 3.5 trillion keys for 12 billion needed** — that's ~290x headroom. This buffer matters because:

1. **Collision avoidance**: If using hash-based generation, you need spare keyspace to handle collisions without excessive retries.
2. **Key wastage**: KGS may waste some keys (allocated but unused due to server crashes). 290x headroom absorbs this easily.
3. **Growth beyond estimates**: If usage doubles or triples, 7 chars still works.

**When you need 8 characters:**

If your scale is 10x higher (1B URLs/month) or your key generation strategy wastes significant keyspace, 8 characters gives you another 62x headroom. The UX cost of one extra character is minimal.

**The tradeoff:**

- Shorter URLs (6 chars): better UX, easier to share verbally, but keyspace pressure means you hit collision problems sooner and can't afford wasteful generation strategies.
- Longer URLs (7-8 chars): negligible UX difference in practice (users copy-paste, not type), but dramatically safer keyspace. Always pick 7 — the UX difference between 6 and 7 characters is invisible in real usage, but the engineering headroom is enormous.

</details>

## High-Level Architecture

<details>
<summary>4. Design the API contract for a URL shortener — what endpoints do you need for creation, redirection, custom aliases, and expiration, what HTTP methods and status codes are appropriate for each, why should the creation endpoint return the full short URL rather than just the key, and how do you handle optional parameters like custom aliases and TTL in the same creation endpoint?</summary>

**Core endpoints:**

```
POST   /api/v1/urls          — Create a short URL
GET    /{shortKey}           — Redirect to the long URL
GET    /api/v1/urls/{shortKey}  — Get URL metadata (optional)
DELETE /api/v1/urls/{shortKey}  — Delete/deactivate a URL (optional)
```

**Creation endpoint:**

```
POST /api/v1/urls
Content-Type: application/json
Authorization: Bearer <token>

{
  "longUrl": "https://example.com/very/long/path?query=value",
  "customAlias": "my-link",     // optional
  "expiresAt": "2027-01-01T00:00:00Z"  // optional
}
```

Response (201 Created):
```json
{
  "shortKey": "a1b2c3d",
  "shortUrl": "https://short.ly/a1b2c3d",
  "longUrl": "https://example.com/very/long/path?query=value",
  "expiresAt": "2027-01-01T00:00:00Z",
  "createdAt": "2026-03-13T10:00:00Z"
}
```

**Why return the full short URL, not just the key:**

The client shouldn't need to know your domain or URL structure. If your domain changes, or you add path prefixes, the client's integration doesn't break. The short URL is the product — return it ready to use.

**Handling optional parameters in one endpoint:**

Custom alias and TTL are optional fields in the same creation request body. The server logic branches:

- If `customAlias` is provided: validate it (length, allowed characters, not taken), use it as the short key. Return 409 Conflict if already taken.
- If `customAlias` is absent: generate a key via KGS or encoding strategy.
- If `expiresAt` is provided: store it with the record. If absent: use a system default (e.g., no expiration, or 5-year default).

**Redirection endpoint:**

```
GET /{shortKey}
→ 302 Found (or 301 Moved Permanently)
   Location: https://example.com/very/long/path?query=value
```

**Status codes summary:**

| Scenario | Status Code |
|----------|-------------|
| URL created | 201 Created |
| Redirect | 302 Found (or 301) |
| Short key not found | 404 Not Found |
| Custom alias taken | 409 Conflict |
| Invalid input | 400 Bad Request |
| URL expired | 410 Gone |
| Rate limit exceeded | 429 Too Many Requests |

**Separation of concerns**: The redirect endpoint (`GET /{shortKey}`) lives on the main domain for minimal URL length. The management API (`/api/v1/urls/...`) can live on a separate subdomain or path prefix — it's a different traffic pattern (authenticated, lower volume).

</details>

<details>
<summary>5. Compare base62 encoding of an auto-incrementing ID vs MD5/SHA256 hash truncation for generating short URLs — how does each approach work, what are the collision characteristics of each, and why does hash truncation become problematic at scale while base62 has a different set of weaknesses around sequential predictability and distributed coordination?</summary>

**Base62 of auto-incrementing ID:**

How it works: A counter (database sequence, distributed ID generator like Snowflake) produces a unique integer. You convert it to base62 to get the short key.

```typescript
function toBase62(num: number): string {
  const chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  let result = '';
  while (num > 0) {
    result = chars[num % 62] + result;
    num = Math.floor(num / 62);
  }
  return result;
}
// toBase62(1000000) → "4c92"
```

- **Collisions**: Zero — every integer is unique, so every base62 output is unique.
- **Weaknesses**:
  - **Predictability**: Sequential IDs mean an attacker can enumerate URLs. `short.ly/a1b2c3d` implies `short.ly/a1b2c3e` exists. This leaks information about creation volume and lets attackers scrape URLs.
  - **Distributed coordination**: In a multi-server setup, you need a centralized counter or range-based allocation (server A gets IDs 1-1M, server B gets 1M-2M). This adds coordination overhead or wastes ID ranges.

**MD5/SHA256 hash truncation:**

How it works: Hash the long URL, take the first 7 characters of the base62-encoded hash as the short key.

```typescript
import crypto from 'node:crypto';

function hashToShortKey(longUrl: string): string {
  const hash = crypto.createHash('md5').update(longUrl).digest('hex');
  // Convert first 12 hex chars to a number, then base62-encode and take 7 chars
  const num = parseInt(hash.substring(0, 12), 16);
  return toBase62(num).substring(0, 7);
}
```

- **Collisions**: Guaranteed at scale. By the birthday paradox, with a 7-character base62 keyspace (~3.5 trillion), collisions become likely well before you fill the space. With 12 billion URLs, the collision probability is non-trivial.
- **Weaknesses**:
  - **Collision handling**: Every write needs a uniqueness check + retry with a modified input (append counter, add timestamp). This adds latency and complexity to the write path.
  - **Same URL, same hash**: The same long URL always produces the same short key — which sounds like deduplication but breaks when different users want separate short URLs for the same destination.
  - **Collision rate grows**: As the database fills, collision probability increases, making retries more frequent. This is a ticking time bomb at scale.

**Summary:**

| | Base62 + Counter | Hash Truncation |
|---|---|---|
| Collisions | None | Inevitable at scale |
| Predictability | Sequential (exploitable) | Unpredictable |
| Coordination | Centralized counter needed | Stateless (no coordination) |
| Write path | Simple (get ID, encode) | Complex (hash, check, retry) |
| Deduplication | No (different ID each time) | Natural (same URL → same hash) |

Neither is ideal at scale, which is why KGS (covered in question 6) tends to win.

</details>

<details>
<summary>6. What is a pre-generated key service (KGS) and why does it tend to win over base62 and hash-based approaches at scale — how does KGS avoid the collision and coordination problems of the other strategies, what infrastructure complexity does it introduce, and what is its performance profile on the write path compared to inline key generation?</summary>

**How KGS works:**

A dedicated service pre-generates millions of unique random short keys offline and stores them in a database. When an application server needs a key to create a new short URL, it requests one (or a batch) from KGS. The key is moved from "unused" to "used" atomically.

**Why KGS wins at scale:**

1. **Zero collisions**: Keys are generated and deduplicated ahead of time. When you hand one out, it's guaranteed unique — no check-and-retry on the write path.
2. **No coordination between app servers**: Each server grabs a batch of keys from KGS and uses them locally. No distributed counter, no range allocation, no consensus protocol.
3. **No predictability**: Keys are random, not sequential. An attacker can't enumerate URLs or infer creation volume.
4. **Decoupled generation from serving**: Key generation happens offline/async. The write path is just "grab a pre-allocated key and insert" — the fastest possible write.

**Infrastructure complexity it introduces:**

- **KGS as a service**: Needs to be highly available — if KGS is down, no new URLs can be created. Requires redundancy (active-passive or active-active with partitioned key ranges).
- **Key storage**: A table of pre-generated keys. At 7 bytes per key, 1 billion pre-generated keys = ~7 GB. Trivial storage.
- **Batch allocation**: App servers request batches (e.g., 1,000 keys at a time) to reduce round trips. Need to handle the case where a server crashes with allocated-but-unused keys (small wastage, acceptable with 3.5 trillion keyspace).
- **Monitoring**: Need to alert when unused key pool drops below a threshold to trigger generation of more keys.

**Write path performance comparison:**

| Strategy | Write path steps | Latency |
|----------|-----------------|---------|
| Base62 + counter | Get next ID → encode → insert | Fast, but counter is a bottleneck |
| Hash truncation | Hash → check uniqueness → retry on collision → insert | Variable (collisions add retries) |
| KGS | Grab pre-allocated key → insert | Fastest and most predictable |

With KGS, the write path is a single DB insert with a known-unique key. No uniqueness check needed (the key is guaranteed unused), no retries, no coordination. The latency is bounded and predictable.

**The tradeoff is operational**: you're adding a new service to manage. But at scale, the simplicity it brings to the write path and the elimination of collision/coordination problems more than justify it.

</details>

<details>
<summary>7. Why might you choose a relational database (PostgreSQL) vs a NoSQL store (DynamoDB, Cassandra) for the URL mapping — given that the access pattern is simple key-value lookups with a heavy read bias, what are the tradeoffs in terms of consistency guarantees, scaling characteristics, operational complexity, and how the choice changes depending on whether you need analytics joins or just fast lookups?</summary>

**The access pattern is the deciding factor.** URL shortener reads are point lookups by short key — the simplest possible query. Writes are single-row inserts. No joins, no range scans, no transactions across rows for the core path.

**NoSQL (DynamoDB / Cassandra) — the natural fit for the core mapping:**

- **Scaling**: Horizontal scaling is built in. DynamoDB auto-scales; Cassandra scales by adding nodes. No manual sharding.
- **Latency**: Single-digit millisecond reads at any scale for point lookups. This is what these systems are optimized for.
- **Operational simplicity at scale**: DynamoDB is fully managed — no replica configuration, no connection pool tuning, no vacuum. Cassandra requires more ops but scales linearly.
- **Consistency**: DynamoDB offers eventual consistency by default (fine for reads — a URL created 100ms ago not being immediately readable is acceptable). Strong consistency available per-request when needed.
- **Cost**: Pay-per-request pricing (DynamoDB) aligns well with variable traffic patterns.

**Relational (PostgreSQL) — when you need more than lookups:**

- **Analytics**: If you need to join URL data with click analytics, user data, or run ad-hoc queries, PostgreSQL's query flexibility is valuable. `SELECT short_key, count(*) FROM clicks GROUP BY short_key` is trivial in SQL, painful in DynamoDB.
- **Consistency**: Strong consistency by default. If "create URL then immediately share it" is a hard requirement with zero tolerance for lag, Postgres is simpler.
- **Scaling ceiling**: Single-node Postgres handles ~10K reads/sec easily. With read replicas, ~50-100K reads/sec. Beyond that, you need manual sharding (Citus, application-level), which adds complexity.
- **Operational burden**: Connection pooling, vacuuming, replica lag monitoring, backup management.

**Practical recommendation:**

For a URL shortener at scale (millions of URLs, thousands of reads/sec), **DynamoDB or a similar NoSQL store for the URL mapping** is the pragmatic choice. The access pattern is pure key-value, and NoSQL excels here without the scaling complexity.

If you need analytics, use a **separate store** — stream click events to an analytics database (ClickHouse, Redshift, or even Postgres). Don't force the URL mapping store to serve both patterns. This separation lets you optimize each store for its workload.

The hybrid approach is what most production URL shorteners use: NoSQL for the hot path (redirects), relational/OLAP for the cold path (analytics, admin queries).

</details>

<details>
<summary>8. Design the database schema for the URL mapping table — what columns are needed beyond the short key and long URL (think metadata, timestamps, ownership), why do you need specific indexes and what types would you choose, and how do schema decisions change depending on whether you chose a relational database vs a NoSQL store?</summary>

**Relational schema (PostgreSQL):**

```sql
CREATE TABLE urls (
  short_key    VARCHAR(8) PRIMARY KEY,  -- the short URL identifier
  long_url     TEXT NOT NULL,            -- original URL
  user_id      UUID,                     -- who created it (nullable for anonymous)
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at   TIMESTAMPTZ,             -- null = never expires
  click_count  BIGINT DEFAULT 0,        -- denormalized for quick display
  is_active    BOOLEAN DEFAULT TRUE     -- soft delete / disable
);

-- For user dashboard: "show me all my URLs"
CREATE INDEX idx_urls_user_id ON urls (user_id) WHERE user_id IS NOT NULL;

-- For expiration cleanup job
CREATE INDEX idx_urls_expires_at ON urls (expires_at) WHERE expires_at IS NOT NULL;

-- For deduplication (optional): find if this long URL was already shortened
CREATE INDEX idx_urls_long_url ON urls USING hash (long_url);
```

**Why these indexes:**

- **Primary key on `short_key`**: The redirect path does `SELECT long_url FROM urls WHERE short_key = ?` — this must be a primary key lookup (O(1) with B-tree or hash).
- **`user_id` index**: Partial index (only non-null) because most queries are "show URLs for user X." Without it, the user dashboard does a full table scan.
- **`expires_at` index**: Partial index for the cleanup job that periodically deletes expired URLs. The job queries `WHERE expires_at < NOW() AND expires_at IS NOT NULL`.
- **Hash index on `long_url`**: If you want deduplication ("same long URL returns same short URL"), you need to look up by long URL. Hash index is ideal — equality-only lookups, more compact than B-tree for long strings.

**NoSQL schema (DynamoDB):**

```
Table: urls
  Partition Key: short_key (String)

Attributes:
  long_url (String)
  user_id (String)
  created_at (Number - epoch)
  expires_at (Number - epoch)  ← DynamoDB TTL attribute
  click_count (Number)
  is_active (Boolean)

GSI: user_id-index
  Partition Key: user_id
  Sort Key: created_at
```

**Key differences from relational:**

- **TTL is native**: DynamoDB automatically deletes items when `expires_at` passes. No cleanup job needed — the database handles it. This is a significant operational win.
- **No hash index on `long_url`**: Deduplication by long URL requires a secondary index or a separate lookup table, which adds cost. Most NoSQL designs skip deduplication.
- **GSI for user lookups**: The Global Secondary Index on `user_id` + `created_at` supports the user dashboard query with sorting. DynamoDB charges for GSI storage and write throughput separately.
- **No partial indexes**: DynamoDB doesn't support conditional indexes. The GSI includes all items, even those without a `user_id`.
- **Denormalization is the norm**: `click_count` as a denormalized counter is natural in NoSQL. In Postgres, you might use a separate `click_stats` table to avoid write contention on the URL row.

</details>

<details>
<summary>9. Why is the choice between 301 (permanent) and 302 (temporary) redirects critical for a URL shortener — how does each affect browser caching behavior, CDN caching, analytics accuracy, and the ability to update or expire URLs after creation? What do most production URL shorteners actually use and why?</summary>

**301 Moved Permanently:**

- Browser caches the redirect indefinitely. After the first visit, the browser goes directly to the long URL without hitting your server.
- CDNs cache it aggressively, further reducing traffic to your origin.
- **You lose analytics**: If the browser never contacts your server again, you can't count clicks, track referrers, or capture geographic data.
- **You lose control**: If you update the long URL or expire the short URL, browsers that cached the 301 will continue redirecting to the old destination until cache is cleared.

**302 Found (Temporary):**

- Browser treats the redirect as temporary — it contacts your server on every click.
- CDNs may cache it briefly but respect cache-control headers.
- **Full analytics**: Every click hits your server, so you can track everything.
- **Full control**: You can update the destination, expire the URL, or add A/B testing at any time.

**The tradeoff is stark:**

| Aspect | 301 | 302 |
|--------|-----|-----|
| Server load | Very low (browsers cache) | Higher (every click hits server) |
| Analytics | Broken after first visit | Complete |
| URL updates | Don't take effect for cached users | Immediate |
| URL expiration | Can't enforce for cached users | Works immediately |
| CDN offloading | Excellent | Moderate (with short TTL) |

**What production shorteners use:**

Most use **302** (or 307, its modern equivalent). Bit.ly, TinyURL, and most analytics-focused shorteners use 302 because analytics is a core feature — it's the main value proposition beyond just shortening.

The exception: if you're building a shortener purely for link management with no analytics (e.g., internal tooling where you just want shorter URLs), 301 saves significant server resources.

**Practical middle ground**: Use 302 for the redirect but set `Cache-Control: private, max-age=0` to ensure no intermediate caching. If you want some CDN offloading without losing all analytics, you can use 302 with a short `max-age` (e.g., 60 seconds) — you'll lose per-click granularity but still capture hourly trends while reducing origin load.

</details>

## Component Deep Dives

<details>
<summary>10. Design the caching layer for a URL shortener — why does power-law traffic distribution (a small percentage of URLs get the vast majority of clicks) make caching extremely effective, how do you size the cache (what percentage of URLs to cache and why), and what eviction policy (LRU, LFU, or TTL-based) best fits this access pattern? Walk through the read path showing cache hit vs miss flows.</summary>

**Why power-law makes caching extremely effective:**

URL click traffic follows a Zipfian / power-law distribution: a tiny fraction of URLs (viral links, popular services) receive the vast majority of clicks. Typically, the top 1% of URLs account for ~50% of traffic, and the top 20% account for ~80-90%. This means a relatively small cache captures a disproportionately large share of reads.

**Cache sizing:**

Using the 12 GB figure from Q2 (top 20% of a month's URLs at ~600 bytes each), achievable cache hit rate is 80-90%+. A more aggressive approach: cache top 5% = ~3 GB, still catching 50-60% of traffic. Either way, 10-20 GB of Redis/Memcached handles this — a single node or small 2-3 node cluster for redundancy.

**Eviction policy — LRU is the best fit:**

- **LRU (Least Recently Used)**: Naturally keeps hot URLs in cache and evicts cold ones. Since popular URLs are accessed constantly, they never age out. Unpopular URLs get evicted quickly after their brief spike of activity. This aligns perfectly with power-law traffic.
- **LFU (Least Frequently Used)**: Theoretically better for power-law, but has a practical problem — a URL that was viral last week retains a high frequency count even after traffic dies. LFU is slow to adapt to shifting popularity. Some implementations add time-decay, but this adds complexity.
- **TTL-based**: Good as a secondary policy (set a 24-hour TTL on all cached entries) but insufficient alone — it doesn't prioritize hot URLs over cold ones.

**Recommendation**: LRU with a TTL ceiling (e.g., 24 hours). LRU handles the steady-state well, and the TTL ensures stale entries (updated or expired URLs) don't persist indefinitely.

**Read path — cache hit:**

```
Client → Load Balancer → App Server → Cache (Redis)
                                        ↓ HIT
                                    Return long URL
                                        ↓
App Server ← long URL ← Cache
    ↓
Client ← 302 Redirect (Location: long URL)
```

Latency: ~5-10ms total (network hops + sub-ms cache lookup).

**Read path — cache miss:**

```
Client → Load Balancer → App Server → Cache (Redis)
                                        ↓ MISS
                                    App Server → Database
                                        ↓
                                    Return long URL (or 404)
                                        ↓
                                    Write to Cache (async or sync)
                                        ↓
App Server ← long URL
    ↓
Client ← 302 Redirect (Location: long URL)
```

Latency: ~15-30ms total (cache miss + DB lookup + cache write). The cache write can be fire-and-forget to avoid adding it to the critical path.

**Cache invalidation**: When a URL is updated or deleted, invalidate the cache entry. With TTL as a safety net, even if invalidation fails, the stale entry expires within hours.

</details>

<details>
<summary>11. Design the analytics pipeline for click tracking — how do you capture geographic location, referrer, device type, and timestamp for every redirect without adding latency to the redirect response? Why is async processing essential here, and what queueing or streaming architecture would you use, and how do you handle the eventual consistency between the redirect happening and the analytics data being queryable?</summary>

**Why async processing is essential:**

The redirect response must be as fast as possible — every millisecond matters for user experience. If you synchronously write analytics data (geo lookup, DB insert, aggregation) before returning the 302, you add 20-100ms to every redirect. With 4,000+ redirects/sec, this also creates write pressure on whatever analytics store you use.

The redirect and the analytics write have fundamentally different SLAs: the redirect is latency-critical, the analytics is throughput-critical with tolerance for seconds or minutes of delay.

**Architecture:**

```
Client → App Server → Return 302 immediately
              ↓ (fire-and-forget)
         Publish click event to Kafka/SQS
              ↓
         Analytics Consumer(s)
              ↓
         Enrich (geo, device) → Write to analytics store
```

**What the app server captures at redirect time (zero-cost data):**

- `short_key` — from the URL path
- `timestamp` — server clock
- `IP address` — from request
- `User-Agent` — from request header
- `Referer` — from request header

All of this is already available in the HTTP request — no external calls needed. The app server publishes a raw click event with these fields to a message queue.

**What the analytics consumer does (async enrichment):**

- **Geo lookup**: IP → country/city using MaxMind GeoIP database (local file, sub-ms lookup). No external API call.
- **Device parsing**: User-Agent → browser, OS, device type using a UA parser library (local, sub-ms).
- **Aggregation**: Increment counters (hourly clicks per URL, daily unique visitors, geographic distribution).
- **Write to analytics store**: ClickHouse, TimescaleDB, or DynamoDB — optimized for time-series append-heavy writes.

**Queue choice:**

- **Kafka**: Best for high throughput. Supports replay (re-process click events if analytics pipeline has a bug), partitioning by `short_key` for ordered processing, and multiple consumers (real-time dashboard + batch rollups).
- **SQS**: Simpler operationally if you're on AWS and don't need replay. Good for moderate scale.
- **Kinesis**: AWS-native Kafka alternative with simpler scaling model.

**Handling eventual consistency:**

The analytics data is queryable seconds to minutes after the redirect. This is acceptable because:

- Users checking their dashboard don't need real-time-to-the-second accuracy.
- If you need a "live" click counter, use a Redis counter incremented at redirect time (cheap, fast, not durable) as a hot counter, and reconcile with the durable analytics store periodically.

**Failure handling:** Kafka retains events for consumer replay; SQS re-delivers on timeout. A dead letter queue catches repeatedly failing events. Detailed failure modes are covered in Q20.

</details>

<details>
<summary>12. Walk through the write path for creating a new short URL end-to-end — where can race conditions occur (two requests trying to claim the same key), how do you guarantee uniqueness (database constraint vs distributed lock vs KGS pre-allocation), and why is ID predictability a security concern? What attack does sequential ID exposure enable and how do you mitigate it?</summary>

**Write path end-to-end (with KGS):**

```
1. Client sends POST /api/v1/urls { longUrl: "https://..." }
2. App server validates input (URL format, length limits, rate limit check)
3. App server grabs a key from its local batch (pre-allocated from KGS)
4. App server inserts: INSERT INTO urls (short_key, long_url, ...) VALUES (...)
5. Return 201 Created with the full short URL
```

**Where race conditions occur:**

**Without KGS (hash-based or counter-based):**

- Two requests hash the same long URL → same short key → both try to insert → one fails with a unique constraint violation.
- Two servers request the next counter value simultaneously → if the counter isn't atomic, they get the same ID.
- Custom alias race: two users request the same custom alias at the same time.

**With KGS:**

- Race conditions are eliminated on the key generation side — each key is allocated to exactly one server.
- The only remaining race: custom aliases, where two users request `short.ly/my-brand` simultaneously.

**Guaranteeing uniqueness (three strategies):**

1. **Database unique constraint (simplest)**: Add `UNIQUE` on `short_key`. If a collision occurs, catch the error and retry with a new key. Works with any generation strategy. The database is the source of truth.

2. **Distributed lock (overkill)**: Acquire a lock on the short key before inserting. Adds latency and complexity. Not recommended — the DB constraint does the same job without the distributed systems headache.

3. **KGS pre-allocation (best)**: Keys are guaranteed unique before they're handed out. The DB unique constraint is still there as a safety net, but it should never fire. No retries needed.

**For custom aliases**: Always rely on the database unique constraint + application-level check (`SELECT 1 FROM urls WHERE short_key = ?` before insert). The small race window between check and insert is covered by the unique constraint — if two concurrent requests pass the check, the second insert fails and returns 409 Conflict.

**ID predictability as a security concern:**

If short keys are sequential (base62 of auto-incrementing IDs), an attacker can:

1. **Enumerate URLs**: Iterate through `a1b2c3d`, `a1b2c3e`, `a1b2c3f`, etc. to discover all shortened URLs. This exposes private documents, internal tools, confidential links.
2. **Estimate volume**: The current ID tells them how many URLs have been created.
3. **Predict future URLs**: They can preemptively check what the next URL will be.

**Mitigation:**

- **Use KGS with random keys**: The primary mitigation. Random 7-character keys are unguessable — 3.5 trillion possibilities means brute force is impractical.
- **If stuck with sequential IDs**: Apply a bijective scrambling function (e.g., format-preserving encryption) to the ID before encoding. The output looks random but is still unique and reversible.
- **Rate limiting on the redirect endpoint**: Detect and block enumeration attempts (many 404s from the same IP in rapid succession).

</details>

<details>
<summary>13. Design the KGS internals — why is a two-table approach (unused keys vs used keys) common, how does batching key allocation to application servers reduce round trips and prevent KGS from becoming a bottleneck, and what happens when a key is handed out but the URL creation subsequently fails? How do you handle key exhaustion and what monitoring would you set up to prevent it?</summary>

**Two-table approach:**

```
Table: unused_keys
  key (VARCHAR(8) PRIMARY KEY)

Table: used_keys
  key (VARCHAR(8) PRIMARY KEY)
  allocated_at (TIMESTAMPTZ)
  used_by_server (VARCHAR)
```

**Why two tables?**

- **Atomic allocation**: Move a batch of keys from `unused_keys` to `used_keys` in a single transaction. This guarantees no key is handed out twice, even under concurrent requests from multiple app servers.
- **Clear accounting**: You always know how many keys remain (`SELECT COUNT(*) FROM unused_keys`) and which server used which keys.
- **Idempotent replenishment**: The key generation job inserts into `unused_keys` only — no risk of overwriting used keys.

**How batching works:**

Instead of one key per request, each app server requests a batch (e.g., 1,000-10,000 keys):

```sql
-- Allocate a batch atomically
WITH batch AS (
  DELETE FROM unused_keys
  WHERE key IN (SELECT key FROM unused_keys LIMIT 1000)
  RETURNING key
)
INSERT INTO used_keys (key, allocated_at, used_by_server)
SELECT key, NOW(), 'app-server-03' FROM batch;
```

The app server stores these keys in memory and uses them for subsequent URL creations without contacting KGS. Benefits:

- **Reduced round trips**: 1 KGS call per 1,000 URL creations instead of 1:1.
- **KGS not a bottleneck**: Even at 40 writes/sec, a batch of 1,000 lasts ~25 seconds. KGS handles a request every 25 seconds per server — trivial load.
- **Latency**: URL creation doesn't wait on KGS; the key is already in memory.

**What happens when a key is allocated but URL creation fails?**

The key moves to `used_keys` but no URL is ever stored for it. This is **acceptable key wastage**. With 3.5 trillion possible 7-character keys and 12 billion needed, wasting even millions of keys is negligible. Trying to reclaim them adds complexity (return-to-pool logic, race conditions) for no practical benefit.

**Key exhaustion handling:**

- **Monitoring**: Alert when `unused_keys` count drops below a threshold (e.g., 10 million keys = ~3 days of buffer at 40 writes/sec).
- **Replenishment job**: A background process generates random 7-character base62 strings, checks for uniqueness against `used_keys`, and bulk-inserts into `unused_keys`. This runs continuously or on a schedule.
- **Emergency fallback**: If KGS is truly exhausted (catastrophic failure of replenishment), the app server falls back to hash-based generation with retry — degraded but functional.
- **Capacity planning**: Pre-generate enough keys to last weeks or months. 100M pre-generated keys at 7 bytes each = ~700 MB. Cheap insurance.

**KGS high availability:**

- Run KGS with a primary-replica database.
- Each KGS instance allocates from a partitioned range of the `unused_keys` table to avoid contention.
- If KGS is temporarily down, app servers continue using their local batch — they have minutes to hours of runway.

</details>

<details>
<summary>14. How would you implement URL expiration end-to-end — when a user sets a TTL on a short URL, what happens at read time (check expiration before redirect vs rely on cleanup), what happens at the storage layer (lazy deletion vs scheduled cleanup job vs database-native TTL like DynamoDB's), and what are the tradeoffs of each approach in terms of storage reclamation, read latency, and complexity?</summary>

**Three strategies for handling expiration:**

**1. Read-time check (lazy evaluation) — always needed:**

```typescript
// Uses the Web standard Response API (Cloudflare Workers / Deno / Bun)
async function redirect(shortKey: string): Promise<Response> {
  const url = await cache.get(shortKey) ?? await db.get(shortKey);

  if (!url) return new Response(null, { status: 404 });

  if (url.expiresAt && url.expiresAt < Date.now()) {
    // Optionally trigger async cleanup
    await cache.delete(shortKey);
    return new Response(null, { status: 410 }); // Gone
  }

  return Response.redirect(url.longUrl, 302);
}
```

Regardless of your cleanup strategy, you **must** check expiration at read time. A cleanup job might not have run yet, and you can't redirect to an expired URL. The check is a simple timestamp comparison — effectively zero latency cost.

**2. Scheduled cleanup job:**

A cron job or scheduled task periodically deletes expired records:

```sql
-- Run every hour
DELETE FROM urls WHERE expires_at < NOW() AND expires_at IS NOT NULL;
```

- **Pros**: Reclaims storage predictably, keeps the database lean, allows recycling of short keys back to KGS.
- **Cons**: Between runs, expired URLs still occupy storage. Job needs to be efficient at scale (batch deletes, avoid locking).
- **Optimization**: Use the partial index on `expires_at` (from question 8) so the cleanup query is fast.

**3. Database-native TTL (DynamoDB):**

```
Table attribute: expires_at (Number - epoch seconds)
DynamoDB TTL enabled on: expires_at
```

DynamoDB automatically deletes expired items within ~48 hours of the TTL timestamp. No application code or cron job needed.

- **Pros**: Zero operational overhead, no cleanup job to manage.
- **Cons**: Deletion is not immediate — items may persist up to 48 hours after expiration. You still need the read-time check to avoid serving expired URLs. No control over deletion timing.

**Tradeoff comparison:**

| Approach | Storage reclamation | Read latency | Complexity |
|----------|-------------------|--------------|------------|
| Read-time check only (no cleanup) | None — expired data stays forever | Minimal (one timestamp comparison) | Lowest |
| Read-time check + scheduled cleanup | Periodic (hourly/daily) | Minimal | Medium (cron job to manage) |
| Read-time check + DynamoDB TTL | Within ~48 hours, automatic | Minimal | Lowest operational burden |

**Recommended approach:**

- **DynamoDB**: Use native TTL + read-time check. Simplest and fully automatic.
- **PostgreSQL**: Read-time check + hourly cleanup job. Consider `pg_partman` to partition by creation month — dropping old partitions is faster than row-by-row deletion.

**Cache interaction**: When a URL expires, the cache entry must also be invalidated. The read-time check handles this (delete from cache when expired URL is found). The TTL on cache entries (from question 10) provides a backstop — even without explicit invalidation, the cache entry expires within hours.

</details>

<details>
<summary>15. How do you design the redirect flow to minimize latency — trace a request from DNS resolution through load balancer, application server, cache lookup, potential database fallback, and HTTP redirect response. Where are the latency bottlenecks, what is a realistic p50 and p99 target, and what optimizations (connection pooling, cache placement, keep-alive) shave off the most time?</summary>

**Full redirect flow with latency breakdown:**

```
Step 1: DNS Resolution          ~0ms (cached by browser/OS after first lookup)
Step 2: TCP + TLS Handshake     ~10-30ms (first request only; keep-alive reuses)
Step 3: Load Balancer            ~1-2ms (L4/L7 routing)
Step 4: App Server Processing    ~1ms (parse URL, extract short key)
Step 5a: Cache Hit (Redis)       ~1-2ms (network hop + sub-ms lookup)
Step 5b: Cache Miss → DB         ~5-15ms (network hop + index lookup)
Step 6: HTTP 302 Response        ~1ms (tiny response body)
Step 7: Return through LB        ~1-2ms
```

**Realistic latency targets:**

| Metric | Cache Hit | Cache Miss |
|--------|-----------|------------|
| p50 | 5-10ms | 15-25ms |
| p99 | 15-30ms | 50-80ms |

With a 90%+ cache hit rate, overall p50 is ~5-10ms and p99 is ~30-50ms.

**Latency bottlenecks (in order of impact):**

1. **TLS handshake**: 1-2 RTTs for the first connection. Mitigated by HTTP keep-alive and connection reuse.
2. **Cache miss → DB round trip**: The biggest variable. A cache miss adds 10-20ms.
3. **Cross-region latency**: If the user is in Europe and your servers are in US-East, every hop adds 80-100ms RTT. Multi-region deployment is the only fix.

**Optimizations ranked by impact:**

**1. Connection pooling (high impact):**
```typescript
// Pre-established connections to Redis and DB
// Eliminates per-request connection setup (~5-10ms savings)
const redisPool = new Redis({ maxRetriesPerRequest: 1, lazyConnect: false });
const dbPool = new Pool({ max: 20, idleTimeoutMillis: 30000 });
```

**2. HTTP Keep-Alive (high impact):**
Eliminates TCP+TLS handshake for repeat visitors. The load balancer and app server must both support it. Saves 10-30ms on repeat requests.

**3. Cache placement — co-located with app servers (medium impact):**
If Redis is in the same availability zone as the app server, the network hop is ~0.5ms. Cross-AZ adds ~1-2ms. Cross-region adds ~50-80ms. Keep cache close to compute.

**4. Read-through cache pattern (medium impact):**
On cache miss, the app server reads from DB, writes to cache, then returns. The cache write is fire-and-forget — don't wait for confirmation before returning the 302.

**5. Edge/CDN caching (high impact, with caveats):**
Place a CDN (CloudFront, Cloudflare) in front. Cache 302 responses with a short TTL (60-300 seconds). This offloads the majority of traffic for viral URLs without completely sacrificing analytics (you still get per-minute resolution). The CDN handles TLS termination at the edge, dramatically reducing latency for geographically distributed users.

**6. Minimal response body:**
The 302 response is just headers — no body to render. Ensure the app server doesn't add unnecessary headers, middleware, or processing.

</details>

<details>
<summary>16. How do you handle the case where a hash-based encoding strategy produces a collision — walk through the collision detection and resolution flow, why does truncating MD5/SHA256 to 7 characters make collisions likely at scale, and at what collision rate does this approach become impractical compared to KGS?</summary>

**Why truncation makes collisions likely:**

MD5 produces a 128-bit hash (2^128 possibilities). When you truncate to 7 base62 characters, you reduce the output space to 62^7 = ~3.5 trillion. By the birthday paradox, collisions become probable at roughly the square root of the keyspace — around √(3.5 × 10^12) ≈ 1.87 million URLs.

In practice, the collision probability per insert is approximately N / keyspace (where N is the number of existing URLs):
- At 1 billion URLs: ~0.03% per insert (1 in ~3,500)
- At 12 billion URLs: ~0.34% per insert (1 in ~300)

At 12 billion URLs, roughly 1 in 300 inserts collides. This is problematic at scale — at 40 writes/sec, that's ~12 collisions per minute requiring retries — but it's not "nearly every insert." The real pain is operational: unpredictable write latency from retries compounding under load.

**Collision detection and resolution flow:**

```typescript
async function createShortUrl(longUrl: string): Promise<string> {
  let attempt = 0;
  const maxAttempts = 5;

  while (attempt < maxAttempts) {
    // Append attempt number to make hash different each retry
    const input = attempt === 0 ? longUrl : `${longUrl}:${attempt}`;
    const hash = crypto.createHash('sha256').update(input).digest('hex');
    const shortKey = toBase62(parseInt(hash.substring(0, 12), 16)).substring(0, 7);

    try {
      await db.query(
        'INSERT INTO urls (short_key, long_url) VALUES ($1, $2)',
        [shortKey, longUrl]
      );
      return shortKey; // Success — no collision
    } catch (err) {
      if (isUniqueViolation(err)) {
        attempt++;
        continue; // Collision — retry with modified input
      }
      throw err;
    }
  }

  throw new Error('Max collision retries exceeded');
}
```

**The retry strategy:**

Each retry modifies the input (append counter, timestamp, or random salt) so the hash changes. Common approaches:
- Append attempt number: `longUrl + ":1"`, `longUrl + ":2"`, etc.
- Append random salt: guarantees a different hash but loses deduplication.
- Append user ID + timestamp: unique per user per moment.

**When hash-based becomes impractical:**

| URLs stored | Collision rate per insert | Avg retries | Verdict |
|-------------|-------------------------|-------------|---------|
| 1M | ~0.00003% | ~1.0 | Fine |
| 100M | ~0.003% | ~1.0 | Manageable |
| 1B | ~0.03% | ~1.0 | Starting to notice |
| 10B | ~0.3% | ~1.003 | Significant — many retries |
| 50B+ | >1% | ~1.01+ | Impractical — every insert may need retries |

The threshold where hash-based becomes impractical: when collision rate exceeds ~0.1%, meaning roughly 1 in 1,000 inserts needs a retry. At this point:

- Write latency becomes unpredictable (some writes take 2-5x longer).
- Under high write concurrency, retries compound — a server retrying holds a connection longer, increasing contention.
- You're doing reads (collision checks) on the write path, adding DB load.

**KGS avoids all of this**: zero collisions, zero retries, constant write latency. The operational complexity of running KGS is lower than debugging and monitoring collision rates at scale.

</details>

## Scaling & Failure Handling

<details>
<summary>17. How would you shard the URL database — what shard key makes sense (short URL hash vs range-based), why is the short URL key better than the long URL for sharding, how do you handle hot shards when a viral URL's analytics data concentrates on one shard, and what rebalancing strategy do you use when adding new shards?</summary>

**Why shard by short URL key (not long URL):**

- The primary access pattern is `GET by short_key` — the redirect path. Sharding by `short_key` means every redirect is a single-shard query. If you shard by `long_url`, a redirect requires a scatter-gather across all shards (you have the short key, not the long URL).
- Short keys are fixed-length, uniformly distributed (especially with KGS-generated random keys), and compact. Long URLs are variable-length, non-uniform, and expensive to hash.

**Hash-based sharding (recommended):**

```
shard_id = hash(short_key) % num_shards
```

- **Uniform distribution**: Random short keys produce an even hash distribution across shards. No hotspots from the data itself.
- **Simple routing**: Given a short key, any app server can compute the shard in O(1).
- **Consistent hashing variant**: Use a hash ring instead of modulo to minimize data movement when adding/removing shards.

**Range-based sharding (not recommended for this use case):**

Shard A gets keys `a*-m*`, Shard B gets `n*-z*`. Problem: if keys aren't uniformly distributed across the alphabet, shards become unbalanced. Also, sequential patterns in key generation can create hotspots.

**Handling hot shards:**

The URL mapping itself doesn't create hot shards — reads for a viral URL hit one shard, but a single record lookup is cheap regardless of frequency. The real hotspot risk is the **analytics data**: a viral URL generates millions of click events that all route to the same shard if analytics are co-located with URL data.

Mitigation strategies:
- **Separate analytics from URL mapping**: Different sharding strategy for each. URL mapping shards by `short_key`; analytics shards by `short_key + time_bucket` or uses a dedicated time-series database (ClickHouse) that handles hot keys natively.
- **Caching absorbs read hotspots**: The viral URL is in cache — the shard rarely sees the actual read.
- **Read replicas per shard**: If a shard does get hammered by reads, replicas absorb the load.

**Rebalancing when adding shards:**

With modulo hashing (`hash % N`), adding a shard requires rehashing and migrating ~(N-1)/N of all data. This is disruptive.

With **consistent hashing**:
- Adding a shard only moves ~1/N of the data (from neighboring nodes on the ring).
- Use virtual nodes (each physical shard owns multiple points on the ring) for better balance.
- Migration process: new shard joins, starts accepting writes immediately, gradually pulls historical data from the old shard owner. During migration, reads check both old and new shard.

If using DynamoDB or Cassandra, sharding and rebalancing are handled transparently by the database. This is a significant advantage of managed NoSQL for this use case.

</details>

<details>
<summary>18. Design the distributed caching layer — why does consistent hashing matter for cache node addition/removal, how do you handle cache warming after a cold start or node failure, what is the tradeoff between cache replication (higher availability, more memory) and single-copy caching (less memory, cache miss on failure), and how do you size the cache cluster for a read-heavy workload?</summary>

**Why consistent hashing matters:**

Without consistent hashing, adding or removing a cache node (e.g., `hash(key) % 3` changes to `hash(key) % 4`) invalidates almost every cache entry — keys map to different nodes. The result is a **cache stampede**: all requests suddenly miss cache and hit the database simultaneously.

With consistent hashing:
- Each cache node owns a segment of the hash ring.
- Adding a node only invalidates keys in the segment it takes over from its neighbor — roughly 1/N of total keys.
- Removing a node only affects keys it owned — they remap to the next node on the ring.

**Cache warming strategies:**

**After a cold start (new deployment, full cluster restart):**
- **Passive warming**: Let the cache populate naturally through misses. For a URL shortener with 4,000 reads/sec, the top URLs will be cached within minutes. The database handles the initial burst.
- **Active warming**: Pre-load the top N URLs from the database based on recent click counts. Query `SELECT short_key, long_url FROM urls ORDER BY click_count DESC LIMIT 100000` and load into cache before accepting traffic.

**After a single node failure:**
- With consistent hashing, only that node's keys are lost. Other nodes are unaffected.
- The database absorbs the extra load for the lost segment temporarily.
- If using Redis Cluster, automatic failover to a replica means zero cold-start impact.

**Replication vs single-copy tradeoff:**

| Approach | Availability | Memory cost | Consistency | Best for |
|----------|-------------|-------------|-------------|----------|
| Single-copy | Node failure = cache misses for its keys | 1x | Simple — one source of truth | Cost-sensitive, DB can handle burst |
| Replicated (Redis Cluster with replicas) | Automatic failover, near-zero downtime | 2-3x | Replica lag (milliseconds) | High availability requirements |

**Recommendation for a URL shortener**: Replicated. The cache is the primary defense against database overload. Losing it, even briefly, sends 4,000+ reads/sec to the database. A 2-3x memory cost (24 GB instead of 12 GB) is cheap insurance. Redis Cluster with one replica per primary node gives automatic failover in seconds.

**Sizing the cache cluster:**

1. **Memory**: Using the 12 GB cache figure from Q2, with replication that's ~24 GB. Redis overhead (pointers, hashtable) adds ~50%, so ~36 GB total. A 3-node cluster with 12 GB each handles this.

2. **Throughput**: Redis handles ~100K+ operations/sec per node. With 4,000 reads/sec (10K at peak), a single node is sufficient for throughput. The cluster is for memory capacity and redundancy, not throughput.

3. **Network**: Each cache response is ~600 bytes. At 10K requests/sec = ~6 MB/sec. Well within any network limit.

**Final topology**: 3 primary nodes + 3 replica nodes (Redis Cluster), consistent hashing for key distribution, 12 GB per node, LRU eviction with 24-hour TTL ceiling.

</details>

<details>
<summary>19. Why must the application tier be stateless for a URL shortener and how do you achieve this — what state (if any) do application servers hold, how does statelessness enable horizontal scaling behind a load balancer, and what load balancing strategy (round-robin, least connections, consistent hashing) works best for the read-heavy redirect traffic vs the write traffic?</summary>

**Why stateless:**

Statelessness enables horizontal scaling — any server can handle any request, servers can be added or removed freely, and failures don't lose data. For a URL shortener doing 4,000+ redirects/sec, this is essential.

The alternative (stateful servers) forces sticky sessions, means losing a server loses its state, and makes scaling up require state migration or replication — all unnecessary complexity for a service whose core operation is a simple key lookup.

**Achieving statelessness:**

All persistent state lives in external stores:
- **URL mappings**: database + cache (Redis)
- **User sessions**: JWT tokens (self-contained, no server-side session) or external session store
- **KGS key batches**: the only "state" app servers hold in memory

**The KGS batch is the one exception.** Each app server holds a batch of pre-allocated keys in memory (as covered in Q13). This is acceptable because keys are disposable — a crash wastes them, but no request depends on a specific server having a specific key.

**Load balancing strategies:**

**For redirect traffic (reads — 99% of traffic):**

**Round-robin** is the best default. Every redirect request is independent — any server can handle it. Round-robin distributes load evenly because:
- All requests do the same work (cache lookup → optional DB lookup → 302 response).
- Processing time is nearly identical per request.
- No server affinity needed.

**Least connections** is marginally better if some requests are slower (cache misses vs hits), but the difference is negligible for sub-50ms requests.

**Consistent hashing by short key** (route all requests for the same URL to the same server) could enable local in-process caching, but Redis already handles this. The added complexity of sticky routing isn't worth it.

**For write traffic (URL creation — 1% of traffic):**

Round-robin is also fine. Each write server grabs from its own KGS batch, so there's no coordination between servers. The only consideration: if writes require authentication, the load balancer might terminate TLS and validate JWTs, which adds slightly more CPU than redirects.

**Practical setup:**

```
                    ┌─────────────────────┐
Internet ──────────►│  Load Balancer (L7)  │
                    │  Round-Robin          │
                    └─────┬───┬───┬────────┘
                          │   │   │
                    ┌─────┘   │   └──────┐
                    ▼         ▼          ▼
                 App-1     App-2      App-3
                (stateless, identical, auto-scaling)
                    │         │          │
                    └────┬────┘──────────┘
                         ▼
                  Redis + Database
```

Auto-scaling based on CPU/request count — when redirects/sec exceeds threshold, spin up more identical servers. The load balancer detects new servers via health checks and starts routing traffic immediately.

</details>

<details>
<summary>20. Put the entire design together — draw the complete architecture showing all components (clients, DNS, load balancers, app servers, KGS, cache layer, database, analytics pipeline, abuse prevention) and trace both the write path (URL creation) and read path (redirect) through the system. Identify the single points of failure and explain how each is mitigated.</summary>

**Complete architecture:**

```
                         ┌──────────┐
                         │  Client  │
                         └────┬─────┘
                              │
                         ┌────▼─────┐
                         │   DNS    │
                         └────┬─────┘
                              │
                    ┌─────────▼──────────┐
                    │   CDN / Edge       │ ← Optional: cache 302s with short TTL
                    │   (CloudFront)     │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Load Balancer    │ ← L7, TLS termination, rate limiting
                    │   (ALB / Nginx)    │
                    └──┬─────┬─────┬────┘
                       │     │     │
              ┌────────┘     │     └────────┐
              ▼              ▼              ▼
         ┌─────────┐   ┌─────────┐   ┌─────────┐
         │ App-1   │   │ App-2   │   │ App-3   │  ← Stateless, auto-scaling
         │(KGS     │   │(KGS     │   │(KGS     │     Each holds a batch of
         │ batch)  │   │ batch)  │   │ batch)  │     pre-allocated keys
         └──┬──┬───┘   └──┬──┬───┘   └──┬──┬───┘
            │  │           │  │           │  │
     ┌──────┘  └─────┐    │  │    ┌──────┘  └──────┐
     ▼               ▼    ▼  ▼    ▼                ▼
┌─────────┐    ┌──────────────────────┐    ┌──────────────┐
│  KGS    │    │   Redis Cluster      │    │  Kafka /     │
│ Service │    │   (Cache Layer)      │    │  SQS Queue   │
│ (keys)  │    │   3 primary +        │    │  (Analytics) │
└────┬────┘    │   3 replica nodes    │    └──────┬───────┘
     │         └──────────┬───────────┘           │
     │                    │                       ▼
     │         ┌──────────▼───────────┐    ┌──────────────┐
     │         │   Database           │    │  Analytics   │
     │         │   (DynamoDB or       │    │  Consumers   │
     └────────►│    PostgreSQL +      │    │  (Geo, UA)   │
               │    read replicas)    │    └──────┬───────┘
               └──────────────────────┘           │
                                           ┌──────▼───────┐
                                           │  Analytics   │
                                           │  Store       │
                                           │ (ClickHouse) │
                                           └──────────────┘
```

**Write path (URL creation):**

```
1. Client → POST /api/v1/urls { longUrl, customAlias?, expiresAt? }
2. Load Balancer → routes to any app server (round-robin)
3. App Server:
   a. Validate input (URL format, auth, rate limit)
   b. If customAlias: check DB for uniqueness
   c. If no alias: grab key from local KGS batch
      (if batch exhausted → request new batch from KGS)
   d. INSERT into database (short_key, long_url, metadata)
   e. Optionally write-through to cache
4. Return 201 Created { shortUrl, shortKey, ... }
```

**Read path (redirect):**

```
1. Client → GET /{shortKey}
2. CDN → check edge cache
   a. HIT → return cached 302 (skips origin entirely)
   b. MISS → forward to origin
3. Load Balancer → routes to any app server
4. App Server:
   a. Look up shortKey in Redis cache
      - HIT → get longUrl
      - MISS → query database → populate cache (async)
   b. Check expiration (if expiresAt < now → return 410 Gone)
   c. Fire-and-forget: publish click event to Kafka/SQS
      { shortKey, timestamp, IP, userAgent, referer }
   d. Return 302 Found, Location: longUrl
5. Analytics consumer (async):
   a. Consume click event
   b. Enrich: IP → geo (MaxMind), UA → device/browser
   c. Write to analytics store (ClickHouse)
```

**Single points of failure and mitigation:**

| Component | Failure Impact | Mitigation |
|-----------|---------------|------------|
| **Load Balancer** | All traffic drops | Deploy in HA pair (active-passive). AWS ALB is inherently multi-AZ. DNS failover to secondary LB. |
| **App Servers** | Reduced capacity | Auto-scaling group with min 3 instances across AZs. Health checks remove unhealthy instances. LB reroutes traffic. |
| **KGS** | Can't create new URLs (reads unaffected) | App servers hold key batches — hours of runway. KGS backed by replicated DB. Fallback to hash-based generation. |
| **Redis Cluster** | Cache misses → DB overload | Redis Cluster with replicas — automatic failover in seconds. Even total cache loss is survivable: DB handles 4K reads/sec short-term (with degraded latency). |
| **Database** | Reads fail (after cache miss), writes fail | Multi-AZ deployment (RDS) or DynamoDB (inherently replicated). Read replicas for read scaling. Automated failover. |
| **Kafka / SQS** | Analytics delayed (redirects unaffected) | Kafka: multi-broker cluster with replication factor 3. SQS: inherently durable. Click events buffered in app server memory briefly if queue is unreachable. |
| **DNS** | Nothing resolves | Use a reliable provider (Route 53, Cloudflare). DNS is inherently distributed/redundant. Low TTL enables fast failover. |

**Abuse prevention (not shown in detail above):**

- **Rate limiting** at the load balancer: per-IP and per-API-key limits on URL creation.
- **URL validation**: Block known malicious domains (phishing, malware) using a blocklist service on the write path.
- **Redirect-time scanning**: For high-risk or newly created URLs, check against Google Safe Browsing API before redirecting.
- **CAPTCHA/proof-of-work** for anonymous URL creation to prevent spam abuse.

**Key design decisions recap:**
- KGS for key generation (no collisions, no coordination, unpredictable keys)
- 302 redirects (preserves analytics and control)
- Separate URL mapping store from analytics store (different access patterns)
- Redis cache with LRU + TTL (power-law traffic makes caching highly effective)
- Stateless app servers with round-robin LB (simplest scaling model)
- Async analytics pipeline (zero impact on redirect latency)

</details>
