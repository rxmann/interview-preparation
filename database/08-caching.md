# Database Notes: Caching

> Part of the [Database Notes](index.md) series.

---

## 8. Caching

### Why cache

A Redis lookup takes under a millisecond. A database query with joins and disk reads can take tens or hundreds of milliseconds. Caching stores copies of frequently read data in memory so future requests skip the expensive path entirely. Beyond latency, caching offloads read traffic from the database — if 95% of reads hit the cache, the database handles only 5% of read volume.

The fundamental cost is that a cache is a second copy of data that can go stale. Every caching decision is a trade between freshness and performance.

---

### Caching patterns

#### Cache-aside (lazy loading)

The most common pattern. The application owns the logic entirely:

1. Check the cache for the key.
2. On a **hit** — return the cached value.
3. On a **miss** — query the database, store the result in the cache with a TTL, return the result.
4. On a **write** — delete the affected cache key (not update it — see gotchas).

**What it solves:** Simple to reason about, the app still works if the cache goes down (just slower), and you only cache what gets requested (no wasted space for cold data).

**When to use it:** Default choice for most read-heavy workloads. API responses, user profile lookups, product catalog data.

**Tradeoffs:**
- Cold-start penalty: the first request for each key always hits the database.
- If cache goes down and comes back empty, all traffic flows to the database temporarily — can overload it.
- Application code gets more complex: every read has the cache check logic, every write has the invalidation logic.

---

#### Read-through

The application always talks to the cache. On a miss, the **cache itself** fetches from the database using a configured data loader, stores the result, and returns it.

**What it solves:** Cleaner application code — the app doesn't need to handle misses. The cache is a transparent layer.

**When to use it:** When you want to remove cache logic from application code, and your caching library supports this pattern (e.g., Spring Cache, AWS DAX for DynamoDB).

**Tradeoffs:**
- Requires a cache library that supports pluggable data loaders.
- Less control over what gets cached and when.
- Same cold-start penalty as cache-aside.

---

#### Write-through

Every write goes **through the cache to the database synchronously**. Both are updated together before returning to the caller.

```
write(key, value)
  → update cache
  → update database
  → return success (after both confirm)
```

**What it solves:** Cache is always consistent with the database for any data that has ever been written. No stale reads immediately after a write.

**When to use it:** When read-after-write consistency matters and you can tolerate slightly higher write latency. Good paired with read-through.

**Tradeoffs:**
- Every write is slower because it waits for both the cache and the database.
- Writes populate the cache even for data that may never be read — wastes memory.
- If the cache goes down, writes fail unless you handle the fallback explicitly.

---

#### Write-back (write-behind)

Writes go to the **cache only** and are asynchronously flushed to the database later, often batched.

```
write(key, value)
  → update cache immediately (fast)
  → queue flush job
  → database updated seconds later in background
```

**What it solves:** Absorbs write bursts well. Applications see instant write acknowledgment. The database sees a smooth, batched stream instead of every individual write. Great for high-frequency updates like counters, view counts, real-time analytics.

**When to use it:** Write-heavy workloads where you can tolerate losing a few seconds of data on a crash. Never for financial transactions, order records, or anything requiring durable acknowledgment.

**Tradeoffs:**
- **Data loss risk:** If the cache node crashes before flushing, those buffered writes are gone.
- More complex failure handling — you need to know what was flushed and what wasn't.
- Reads from another node may see stale data until the flush happens.

---

#### Write-around

Writes go **directly to the database, bypassing the cache**. Reads still check the cache first (lazy population).

**What it solves:** Avoids polluting the cache with data that is written once and rarely read (bulk imports, batch jobs, log writes). The cache stays full of hot read data instead of one-time writes.

**When to use it:** When you have a mix of bulk-write workloads (ETL, imports) alongside a hot read cache. Prevents a large batch job from evicting all your cached product pages.

---

### Cache invalidation strategies

**TTL (time-to-live):** Every key expires after a fixed duration. Self-healing and zero coordination needed. The tradeoff is data can be stale for up to TTL duration. Good as a safety net — not as the primary consistency mechanism.

**Explicit deletion on write:** When data changes, immediately delete the affected cache key. The next read repopulates it from the database (lazy). More precise than TTL, but you must track every key a given write affects. If your user object is cached under `user:{id}`, `session:{id}`, and `profile:{id}`, a user update must delete all three.

**Event-driven invalidation:** A database CDC event (e.g., via Debezium) or an application event triggers cache cleanup. The cache invalidation is decoupled from the write path. Scales well, but adds infrastructure complexity and there is a small window between the write and the cache invalidation where stale data can be served.

**Versioned keys:** Embed a version number in the cache key — `user:v3:42`. Bumping the version makes all old keys unreachable (they'll expire via TTL) without touching them directly. Useful for cache-busting a large set of related keys at once (e.g., after a schema change).

**Best practice:** Combine explicit deletion with a TTL as a safety net. The deletion gives you near-instant freshness; the TTL cleans up anything you missed.

---

### Cache failure modes

**Cache stampede** (thundering herd): A hot key expires. Thousands of concurrent requests all miss at the same moment and all rush to rebuild the same value from the database simultaneously. The database sees a spike that may be 100× normal query volume.

Fixes:
- **Mutex/single-flight**: Only one request rebuilds the cache; others wait or receive the slightly stale value.
- **Probabilistic early expiration**: Before the key actually expires, start regenerating it with increasing probability as expiry approaches. The key is refreshed before it goes cold.
- **Background refresh**: A job proactively refreshes hot keys before they expire.

**Cache penetration**: Requests for keys that don't exist in the database (and therefore never get cached) hit the database on every request. An attacker can exploit this by querying random IDs to bypass the cache entirely.

Fixes:
- Cache null/empty results with a short TTL (`user:999 → null, TTL 60s`).
- Use a **bloom filter** at the cache layer: a probabilistic data structure that says "this key definitely doesn't exist" (no false negatives) before even hitting the database.

**Cache avalanche**: Many keys expire simultaneously (or the entire cache goes down), flooding the database with misses all at once.

Fixes:
- **TTL jitter**: Randomize expiry times (`TTL = base + rand(0, 300s)`) so expirations are spread out over time.
- Highly available cache setup (Redis Cluster, Redis Sentinel).
- Circuit breakers to protect the database if miss rate spikes — return a degraded response rather than overloading the database.

---

### Eviction policies

When the cache fills up, something must go. How you choose what to evict determines how well the cache performs under memory pressure.

- **LRU (least recently used):** Evicts the key that hasn't been accessed the longest. Good general default — it keeps recently active data.
- **LFU (least frequently used):** Evicts the key accessed the fewest times overall. Better for workloads with stable hotspot patterns. Resistant to a one-off full table scan flushing all your hot keys.
- **TTL-based:** Evicts keys closest to expiry first.
- **Random:** Simple, low overhead. Works surprisingly well in practice for uniform access patterns.
- **allkeys-lru vs volatile-lru** (Redis): `allkeys-lru` evicts any key; `volatile-lru` evicts only keys with a TTL set, leaving TTL-less keys untouched.

---

### Redis vs Memcached

**Redis** supports rich data structures: strings, hashes, lists, sets, sorted sets, bitmaps, streams. It has optional persistence (RDB snapshots, AOF log), replication, clustering, pub/sub, Lua scripting, and atomic multi-step operations via transactions. This makes it useful far beyond caching — rate limiting, job queues, leaderboards, session stores.

**Memcached** is a simpler, multi-threaded string cache. No persistence, no HA built in, no data structures beyond key-value. It's slightly faster per core for pure string caching workloads because of its simpler architecture.

Redis is the default choice for almost everything today. Memcached only makes sense when you need a very large, pure string cache and want to avoid Redis's memory overhead for data structures you won't use.

---

### Gotchas

- **Delete on write, never update in place.** If two concurrent writes race, their cache updates can arrive out of order and permanently leave a stale value. Deleting forces the next read to fetch fresh from the database.
- **Write-back data loss is real.** Only use it for data where losing seconds of writes is explicitly acceptable (view counts, analytics). Never for orders, payments, or user data.
- **TTL alone is not a correctness strategy.** It bounds staleness but doesn't prevent it. Combine with explicit invalidation for data where freshness matters.
- **Measure hit ratio before adding caching.** A cache with a 20% hit ratio adds latency and operational complexity for little benefit. You want 90%+ for a cache to be worth it.
- **Cascading cache invalidation is fragile.** One object cached under five different keys means five deletions on every write. This often signals that your caching scheme is too granular.
- **Don't cache mutable per-user data at a shared key.** If `user:profile` is cached globally and two users have different permissions, one might see the other's data. Always scope cache keys tightly.
