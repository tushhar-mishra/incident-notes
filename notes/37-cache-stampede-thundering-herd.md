# 37 · Cache Stampede and Thundering Herd

> **Level:** ★★★ advanced
>
> **You will learn:** what happens when **many requests arrive at the same moment** for the same thing: a **cache stampede** (a hot key expires and thousands of requests hit the database together), a **cold cache** after a restart or flush, a **thundering herd** (many waiting clients released at once, many instances reconnecting at once), **synchronized expiry**, and the fixes: **request coalescing**, **locks/single-flight**, **stale-while-revalidate**, **jittered TTLs**, **cache warming** and **admission control**.
>
> **You need:** topics 01–36. Helpful: [SD 14 · Caching](../../System-Design/notes/14-caching.md), [SD 15 · CDN and edge](../../System-Design/notes/15-cdn-and-edge.md).
>
> **Part 2 · Recurring failures · F. Overload and cascades** · Previous: [36 · Retry storms](36-retry-storms.md) · Next: [38 · Cascading failures](38-cascading-failures.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A cache protects the database by answering most requests. The database is sized for the **misses**, not for all traffic. If the cache suddenly stops answering (key expired, cache flushed, cache restarted), **all** traffic goes to the database at once.

```
Normal:   10,000 req/s → cache (99% hits) → 100 req/s → DB   ✅
Stampede: hot key expires → 10,000 req/s miss at the same moment → 10,000 req/s → DB  ❌
Herd:     27,000 clients all waiting on one slow thing → it finishes → all 27,000 fire next requests at once ❌
```

**Picture it.** A shop that opens its doors at 9:00 on Black Friday. The shop can serve 50 people a minute, but 5,000 rush in at the same second. The shop isn't too small for the day; it's too small for **that second**.

**The one thing to know.** Don't let many requests **rebuild the same thing at once**: let **one** request refresh the value while the others **wait or get the stale value**. And never let lots of keys or clients be **synchronized** (same TTL, same restart time).

**Try it now:** look at how one service sets cache TTLs. Are they all exactly the same (e.g. 3600)? Add jitter.

**You've got it if you can:** explain three fixes for a hot-key stampede and how to restart a cache safely.

> 📖 **That's the core.** Everything below is the **full reference**. Real case: [69 · Canva, November 2024](69-canva-2024.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Cache | Hit ratio drops suddenly (e.g. 99% → 20%); after a cache restart/flush/failover; at regular intervals (synchronized TTL expiry) |
| Database | Sudden spike in identical queries; CPU/connection saturation (topic 30) |
| Graphs | Periodic spikes every N minutes (TTL); a huge spike right after a deploy/restart; a spike after a slow dependency **recovers** |
| Latency | Spikes then recovery (lucky) or spikes then collapse (stampede triggers cascade, topic 38) |
| Edge/CDN | Origin request rate jumps after a purge or config change |

---

## Analogy

**Black Friday doors** (Core). **Request coalescing** = one shop assistant fetches the item from the stockroom while everyone else who asked for the same item waits at the counter, instead of 500 people running into the stockroom.

---

## How it happens

| Trigger | Mechanism |
|---|---|
| **Hot key expiry** | A popular key's TTL ends → every request misses → all query the DB for the same value |
| **Synchronized TTLs** | Thousands of keys created at the same time (e.g. after a deploy) with the same TTL → expire together |
| **Cold cache** | Cache restart, flush, failover to an empty replica, new cache cluster, a deploy that changes cache key format |
| **Thundering herd after a slow step** | Clients all blocked on one slow thing (an asset download, a lock, a leader election); when it completes, they all proceed at once — Canva's November 2024 incident is this pattern |
| **Reconnect herd** | Many clients/instances reconnect at the same time after a network blip or a restart (watchers relisting, topic 22) |
| **CDN purge** | Purging everything sends all traffic to origin at once |

---

## How to find it

**First:** did the cache stop answering?

```bash
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses|evicted_keys|expired_keys"
redis-cli INFO server | grep uptime_in_seconds      # did Redis just restart?
```
```
keyspace_hits:91234567
keyspace_misses:48210012
evicted_keys:0
expired_keys:1201
uptime_in_seconds:340              ← restarted 6 minutes ago → cold cache
```
Prometheus: `rate(redis_keyspace_hits_total[1m]) / (rate(redis_keyspace_hits_total[1m]) + rate(redis_keyspace_misses_total[1m]))`.

**Second:** is the DB load from **one** query/key?

```sql
SELECT left(query, 80), calls FROM pg_stat_statements ORDER BY calls DESC LIMIT 5;
```
Or with Redis: `redis-cli --hotkeys` (needs an LFU eviction policy) to see the hottest keys.

**Third:** what caused the miss wave? Restart/flush/failover time, a deploy changing key names, TTL distribution (`redis-cli TTL <key>` on samples — all identical?), CDN purge logs.

---

## How to fix it

### Immediate

- **Shed or limit load to the origin**: rate-limit the expensive endpoint, serve a degraded response, block at the edge (Canva used a firewall rule at their CDN to block traffic so new gateway instances could start, topic 69).
- **Bring traffic back in steps** while the cache refills.
- **Warm the cache** for the top keys before letting full traffic through.
- If a CDN purge caused it, stop further purges.

### Permanent

| Fix | How it helps |
|---|---|
| **Request coalescing / single-flight** | Only one request per key goes to the DB; others wait for its result (Go `singleflight`, Nginx `proxy_cache_lock`, CDN collapsed forwarding) |
| **Lock on refresh** | A short distributed lock (e.g. Redis `SET key NX PX`) so only the lock holder rebuilds |
| **Stale-while-revalidate** | Serve the old value while one background task refreshes it (HTTP `Cache-Control: stale-while-revalidate`, app-level soft TTL) |
| **Probabilistic early refresh** | Refresh a little **before** expiry, with randomness, so hot keys never all expire at once |
| **Jittered TTLs** | `ttl = 3600 + random(0, 600)` → no synchronized expiry |
| **Warm-up on deploy/restart** | Pre-load top keys; slow-start new instances (LB slow-start mode) |
| **Admission control / load shedding** | Protect the origin: reject excess quickly rather than collapse |
| **Caches with replicas/persistence** | Failover to a warm replica, not an empty one |

---

## How to verify the fix

- Hit ratio back to normal; DB query rate back to baseline.
- A test: expire a hot key under load → only **one** DB query appears for it.
- Restart one cache node in staging under load → DB load rises within safe limits.

---

## Prevention

- Alerts on hit ratio drop and origin request-rate spikes.
- Never restart/flush all cache nodes at once; rolling, with warm-up.
- Load tests that include **cold start** scenarios.
- Review TTL settings for synchronization.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a Redis failover promoted an empty replica (persistence off), the hit ratio fell to near zero, and the DB, sized for 1% of traffic, received 100%. Root cause: the system depended on a warm cache for survival without planning for a cold one.

2. **Q: How did you detect it? Why not earlier?**
   A: DB saturation alerts. Hit ratio wasn't alerted on, and no one had tested a cold-cache scenario.

3. **Q: What did you check first?**
   A: Cache hit ratio and Redis uptime, then whether DB load was dominated by one query — that confirms a stampede.

4. **Q: How did you stop impact?**
   A: Rate-limited the heaviest endpoints, served stale/degraded content, and admitted traffic in steps while the cache warmed.

5. **Q: Permanent fix?**
   A: Single-flight on cache misses, jittered TTLs, stale-while-revalidate, replica persistence, and a warm-up procedure.

6. **Q: How did you confirm?**
   A: A staging test with a cold cache at production traffic: the DB stayed under 60% and latency recovered in minutes.

7. **Q: What if shedding didn't help — the DB stayed overloaded?**
   A: The stampede may have triggered a retry storm or pool exhaustion (topics 36, 30). Cut retries, shed more, kill stuck queries, and restore traffic gradually.

8. **Q: How is a thundering herd different from a cache stampede?**
   A: A stampede is many requests missing the cache for the same key. A thundering herd is more general: many waiters released at the same moment (after a lock, a slow fetch, a restart), all hitting the next step together. The fixes overlap: spread work in time and coalesce identical work.

9. **Q: What would you do differently?**
   A: Design for cold start from day one: capacity plans assume the cache can be empty.

10. **Q: What if it happened during a deploy?**
    A: Deploys often change cache keys or restart caches; combine that with new code and you get a stampede plus a possible bug. Version keys carefully, warm before switching, and roll out gradually.
