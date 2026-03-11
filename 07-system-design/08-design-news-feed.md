# System Design: News Feed

> **26 questions**

- Requirements and estimation: feed reads vs post writes ratio, fan-out write amplification per post, storage and throughput estimates
- API design: publish post, fetch feed, follow/unfollow, cursor-based pagination
- Data models: users, posts, follows, feed entries, social graph
- High-level architecture: post service, fan-out service, feed service, social graph service
- Fan-out on write (push) vs fan-out on read (pull): latency, storage, consistency tradeoffs
- Hybrid fan-out: follower count threshold, merging pushed and pulled content at read time
- Celebrity problem (hot key / fan-out explosion): why pure push breaks, hybrid solution
- Social graph storage: relational adjacency list vs graph database vs Redis sets
- Feed ranking: signals (recency, engagement, affinity), relevance scoring, write-time vs read-time ranking tradeoffs
- Content filtering: spam detection, muted/blocked user filtering, content policy enforcement, where filtering happens in the pipeline
- Notifications on new posts: push notification triggers during fan-out, decoupling notification delivery from feed writes, deduplication
- Real-time feed updates: polling vs long polling vs SSE vs WebSockets, cost of push at scale
- Fan-out service: follower lookup, batched writes, worker distribution, partial failure handling
- Feed cache: Redis data structure (sorted set, list), post IDs vs full objects, sizing and eviction
- Feed read path: cache lookup, celebrity post merge, post hydration, ranking, cursor-based pagination, cache miss
- Post updates and deletes: propagation through fan-out-on-write without re-fanning
- Post storage: database selection, media handling, indexes for ID/user/time-range queries
- Cold start and cold feed: initial feed generation for new users, aggregating from many accounts, cache pre-warming
- Scaling fan-out: viral posts, worker scaling, queue depth, backpressure, active vs inactive user priority
- Partitioning and sharding: feed cache by user ID, post storage by post ID, hot shard handling
- Cache failure: preventing database stampede, request coalescing, circuit breaking, degraded feed
- Multi-region: social graph replication, fan-out region placement, consistency tradeoffs
- Observability: feed freshness SLI (post-to-feed latency), fan-out lag monitoring, queue depth alerting, cache hit rate tracking

---

## Requirements & Estimation

<details>
<summary>1. How would you scope the requirements for a news feed system — what are the key read vs write ratios, how do you estimate the fan-out work per post, and why does the read-heavy nature of feeds fundamentally shape every architecture decision that follows?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Design the API for a news feed system covering publish post, fetch feed, and follow/unfollow — why is cursor-based pagination essential for feeds instead of offset-based, what fields does each endpoint need, and how do you handle the edge cases (empty feeds, deleted posts in a page, real-time insertions shifting the cursor)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What data models does a news feed system need — design the schemas for users, posts, follows, feed entries, and the social graph, explaining why the follow relationship and feed entry table are separate from the posts table, and how the social graph structure affects fan-out performance?</summary>

<!-- Answer will be added later -->

</details>

## High-Level Architecture

<details>
<summary>4. Walk through the high-level architecture of a news feed system — what are the responsibilities of the post service, fan-out service, feed service, and social graph service, why do you separate them instead of building a monolith, and how does a post flow from creation to appearing in a follower's feed?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. Explain the fundamental tradeoff between fan-out on write (push) and fan-out on read (pull) — how does each approach work, what are the latency, storage, consistency, and write amplification tradeoffs, and under what conditions does each model break down?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why does a hybrid fan-out approach exist and how does it work — what follower count threshold triggers the switch from push to pull, how do you merge pushed and pulled content at read time, and what complexity does the hybrid model add that pure push or pure pull avoids?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What is the celebrity problem (hot key / fan-out explosion) and why does it break a pure fan-out-on-write system — walk through the math of what happens when a user with 10 million followers posts, how does the hybrid solution solve this, and what other mitigations exist beyond simply switching to pull for high-follower accounts?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. How would you store the social graph (follow relationships) and why does the choice matter for fan-out performance — compare a relational adjacency list, a graph database, and Redis sets in terms of query patterns (get all followers, get all following, check if A follows B), write performance, and operational complexity?</summary>

<!-- Answer will be added later -->

</details>

## Component Deep Dives

<details>
<summary>9. How does feed ranking work — what signals (recency, engagement, affinity, content type) feed into a ranking model, how do you compute a relevance score, and what is the tradeoff between write-time placement (pre-ranked during fan-out) vs read-time ranking (scored on fetch)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How do you handle content filtering in the feed — where do spam detection, muted/blocked user filtering, and content policy enforcement happen in the pipeline, why is filtering at read time sometimes necessary even when you fan out at write time, and how do you avoid re-fanning the entire feed when a user blocks someone?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How would you deliver real-time feed updates to connected clients — compare polling, long polling, SSE, and WebSockets for this use case, what are the cost and scaling implications of maintaining persistent connections at scale, and when is simple polling actually the better choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Walk through the internals of the fan-out service — how does it look up a poster's followers, batch the writes to individual feed caches, distribute work across multiple workers, and handle partial failures where some followers' feeds are updated but others fail mid-batch?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. How would you design the feed cache layer — why use Redis, what data structure (sorted set vs list) works best and why, should you store full post objects or just post IDs in the cache, how do you size the cache (how many posts per user), and what eviction strategy handles the long tail of inactive users?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. Walk through the feed read path for a normal user — starting from a cache lookup, how do you hydrate post IDs into full post objects, merge in celebrity posts that were pulled at read time rather than pushed, and return a ranked, filtered page of results using cursor-based pagination?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. What happens when a user's feed cache is completely empty (cache miss) — how do you reconstruct their feed on the fly, what is the performance cost compared to a cache hit, and what strategies (pre-warming, lazy population, fallback to a global trending feed) prevent cold cache reads from degrading the user experience?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How do you handle post updates and deletes in a fan-out-on-write system — when a user edits or deletes a post that has already been fanned out to millions of feeds, how do you propagate that change without re-fanning, what consistency guarantees can you realistically offer, why does storing only post IDs in the feed cache simplify this problem, and what tombstone or soft-delete mechanism prevents deleted posts from reappearing during cache rebuilds?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How would you design post storage — what database fits best for posts (relational vs NoSQL), how do you handle media (images, videos) associated with posts, and what indexes do you need to support efficient queries by post ID, by user ID (all posts from a user), and by time range?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. How do you handle cold start and cold feed scenarios — when a brand new user signs up and follows 200 accounts, how do you generate their initial feed without overloading the system, how do you aggregate posts from many accounts efficiently, and what cache pre-warming strategies prevent their first feed load from being painfully slow?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. How do you trigger push notifications during fan-out without coupling notification delivery to feed writes — when a user posts, how does the fan-out service decide which followers should receive a push notification vs just a feed update, how do you decouple the notification pipeline so a slow notification service doesn't block feed writes, and how do you handle deduplication when the same post triggers notifications through multiple paths (fan-out, reshare, mention)?</summary>

<!-- Answer will be added later -->

</details>

## Scaling & Failure Handling

<details>
<summary>20. How do you scale the fan-out system when a viral post hits — what happens to worker load and queue depth when a high-follower account posts content that also gets reshared widely, how do you implement backpressure to prevent the fan-out queue from overwhelming downstream services, and why should you prioritize active users over inactive ones during fan-out?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. How would you partition and shard the news feed system — why shard the feed cache by user ID and post storage by post ID, what problems does each sharding key solve, and how do you handle hot shards (e.g., a celebrity's post storage shard or a viral post being read from millions of feed caches on the same shard)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. What happens when the feed cache layer fails — how do you prevent a database stampede when millions of users suddenly hit the post database directly, what role do request coalescing and circuit breaking play, and how do you serve a degraded but functional feed while the cache recovers?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. How would you deploy a news feed system across multiple regions — how do you replicate the social graph so fan-out can happen close to users, where do you place fan-out workers relative to the user's home region vs the poster's region, and what consistency tradeoffs are acceptable (e.g., a follower in another region seeing a post a few seconds later)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. A user reports that new posts from accounts they follow are not appearing in their feed for several minutes — walk through how you would diagnose this end-to-end: checking the fan-out queue depth, verifying the social graph has the follow relationship, inspecting the user's feed cache, and determining whether the issue is fan-out lag, a cache miss, or a ranking/filtering bug that's suppressing the content.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Your fan-out queue depth is growing unboundedly and workers can't keep up after a celebrity with 50 million followers posts — walk through how you handle this in real time: what circuit breakers activate, how do you switch that post to pull-based delivery, how do you prioritize which followers get the update first, and what metrics would you monitor to detect this situation before it cascades?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. How would you design observability for a news feed system — what SLI would you use to measure feed freshness (post-to-feed latency), how do you monitor fan-out lag and set alerting thresholds on queue depth, what does a healthy cache hit rate look like and when does a drop signal a problem, and how do these metrics help you distinguish between a fan-out bottleneck, a cache failure, and a ranking bug?</summary>

<!-- Answer will be added later -->

</details>
