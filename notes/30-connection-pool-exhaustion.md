# 30 · Connection-Pool Exhaustion

> **Level:** ★★ intermediate
>
> **You will learn:** what a **connection pool** is, the **math** of pools across many pods (pods × pool size vs database `max_connections`), the three ways a pool runs dry (**slow queries**, **leaked connections**, **too many clients**), why it looks like "**the app is slow**" before it looks like errors, **Little's Law**, the commands to see connections on PostgreSQL/MySQL, and fixes from **PgBouncer** to **timeouts**.
>
> **You need:** topics 01–29. Helpful: [SD 16 · Database fundamentals](../../System-Design/notes/16-database-fundamentals.md), [SD 31 · Capacity planning (Little's Law)](../../System-Design/notes/31-capacity-planning-and-performance-testing.md).
>
> **Part 2 · Recurring failures · E. Data** · Previous: [29 · Load balancer and health checks](29-load-balancer-and-health-checks.md) · Next: [31 · Slow queries, locks and migrations](31-slow-queries-locks-and-migrations.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Opening a database connection is slow and uses server memory, so apps keep a **pool** of open connections and lend them to requests. If every connection is busy, new requests **wait** for one. If they wait too long, they fail.

```
Pool (size 10)  [■■■■■■■■■■]  all in use
Requests waiting: ●●●●●●●●●●●●●●●●  → "timeout acquiring connection after 30000ms"

Database max_connections = 500
Pods: 60 × pool 10 = 600 possible connections  → database rejects: "too many connections"
```

**Picture it.** A taxi rank with 10 taxis. If each ride suddenly takes 10× longer (slow query), the rank empties and a queue of people forms. If drivers park and go home without returning (leak), the rank empties too. If 60 hotels each send 10 people at once, the city's taxi company (DB) runs out of taxis.

**The one thing to know.** **Pool exhaustion is usually a symptom.** Something made connections **held longer** (slow queries, locks, a slow dependency called while holding a connection) or made **more clients** (scale-out). Find what changed the hold time.

**Try it now:** Little's Law: connections in use = requests per second × seconds each request holds a connection. 200 req/s × 0.05 s = 10 connections. If queries slow to 0.5 s → 100 needed.

**You've got it if you can:** do the pool math for a deployment and find what's holding connections on PostgreSQL.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: GitHub's 2026 reports (topic 73).

---

## Symptoms

| Where | What you see |
|---|---|
| App logs | `HikariPool-1 - Connection is not available, request timed out after 30000ms`, `QueuePool limit of size 5 overflow 10 reached, connection timed out` (SQLAlchemy), `sorry, too many clients already` / `FATAL: remaining connection slots are reserved` (PostgreSQL), `Too many connections` (MySQL error 1040) |
| Latency | p99 latency climbs to exactly the pool **acquire timeout** (e.g. 30 s), then errors |
| Metrics | Pool active = max; pending/waiting threads > 0; DB connection count at `max_connections` |
| Pattern | Starts at peak, after scale-out (more pods), after a deploy (new query, leak), or when the DB slows down |
| Users | Slow pages then 500s/504s |

---

## Analogy

The **taxi rank** (Core).

---

## How it happens

### Pool math

```
Total possible connections = number of app instances × max pool size (per instance)
                             (+ background workers, cron jobs, migrations, admin tools, replicas)
Must stay below DB max_connections (minus reserved superuser connections)
```

- HPA scales from 20 → 60 pods during a spike → connections triple → DB limit hit → **scaling out made it worse**.
- Each PostgreSQL connection is a **separate server process** with its own memory; thousands of connections hurt the DB even before the limit.

### Why connections are held too long

| Cause | Example |
|---|---|
| **Slow queries** | Missing index after a data growth; a plan flip (topic 31, Clerk 2026 topic 72) |
| **Locks** | A migration or a long transaction blocks others; they hold connections while waiting |
| **Slow external call inside a transaction** | Code opens a transaction, calls a payment API (2 s), then commits → connection held for 2 s |
| **Leaks** | Code paths that never return the connection (exceptions, missing `close()`/`finally`) |
| **Idle in transaction** | App begins a transaction and forgets to commit/rollback → PostgreSQL state `idle in transaction` |

---

## How to find it

**First:** confirm on both sides: app pool metrics and DB connection count.

App (Prometheus examples): `hikaricp_connections_active`, `hikaricp_connections_pending`, `hikaricp_connections_max`. Pending > 0 for minutes = exhaustion.

```sql
-- PostgreSQL: who is connected, and what are they doing?
SELECT usename, application_name, client_addr, state, count(*)
FROM pg_stat_activity
GROUP BY 1,2,3,4 ORDER BY 5 DESC LIMIT 10;
```
```
 usename | application_name | client_addr | state               | count
---------+------------------+-------------+---------------------+-------
 app     | checkout         | 10.0.4.12   | idle in transaction |   140
 app     | checkout         | 10.0.4.13   | active              |    85
 app     | reports          | 10.0.9.20   | active              |    60
```
- **140 `idle in transaction`** → transactions opened and not closed: leak or slow work inside a transaction.

```sql
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
```

**Second:** why are connections held? Longest-running queries and locks:

```sql
SELECT pid, now() - xact_start AS xact_age, state, wait_event_type, left(query, 80)
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY xact_age DESC NULLS LAST LIMIT 10;

-- Who is blocking whom
SELECT pid, pg_blocking_pids(pid) AS blocked_by, left(query, 60)
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
```
- One old transaction blocking 200 others → lock pile-up (topic 31).

MySQL: `SHOW PROCESSLIST;`, `SHOW STATUS LIKE 'Threads_connected';`, `SHOW VARIABLES LIKE 'max_connections';`.

**Third:** what changed? Pod count (HPA), a deploy (new query/leak), data growth, a new consumer (reports job), a slow dependency called inside transactions.

```bash
kubectl get hpa -n shop
kubectl get deploy -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,REPLICAS:.spec.replicas | sort -k3 -n | tail
```

---

## How to fix it

### Immediate

- **Kill the blockers / idle-in-transaction sessions** (carefully; their work is rolled back):
  ```sql
  SELECT pg_terminate_backend(pid) FROM pg_stat_activity
  WHERE state = 'idle in transaction' AND now() - state_change > interval '5 minutes';
  ```
- **Stop the extra load:** pause the reports job, scale down the non-critical consumer, **cap** HPA max replicas so pods × pool fits.
- **Roll back** if a deploy introduced a leak or slow query.
- Don't just raise `max_connections` blindly: it needs a restart on PostgreSQL and more connections may make the DB slower.

### Permanent

- **Connection pooler** in front of the DB: **PgBouncer** (transaction pooling), RDS Proxy, ProxySQL for MySQL → thousands of app connections share a few hundred DB connections.
- **Size pools from math:** (DB max − reserved) ÷ (max pods + other clients); smaller pools are often faster.
- **Timeouts:** `statement_timeout`, `idle_in_transaction_session_timeout` (PostgreSQL), pool acquire timeout shorter than the request timeout (fail fast).
- Never call external services inside a DB transaction.
- Leak detection in the pool (`leakDetectionThreshold` in HikariCP).
- Separate pools (bulkheads) for critical vs background work.

---

## How to verify the fix

- Pool pending = 0; active well below max at peak.
- DB connections well below `max_connections`; no `idle in transaction` older than seconds.
- p99 latency normal; error rate normal.
- A load test at peak × 1.5 with max HPA replicas stays inside the DB limit.

---

## Prevention

- Alerts: pool pending > 0 for 2 min; DB connections > 80% of max; long transactions > N min.
- HPA max tied to DB capacity (documented).
- Query performance monitoring (`pg_stat_statements`), slow query log.
- Load tests that include scale-out.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a new code path called a payment API inside a DB transaction. When the API slowed to 2 s, each request held a connection for 2 s, the pool emptied and requests queued. Trigger: provider slowness. Root cause: external call inside a transaction, no acquire/statement timeouts.

2. **Q: How did you detect it, and why not earlier?**
   A: Latency SLO burn. Pool pending metrics existed but had no alert.

3. **Q: What did you check first?**
   A: `pg_stat_activity` grouped by state and client, to see whether connections were active, idle-in-transaction, or blocked — each points to a different cause.

4. **Q: How did you stop impact?**
   A: Terminated long idle-in-transaction sessions, disabled the feature by flag, and capped HPA to stop adding connections.

5. **Q: Permanent fix?**
   A: Moved the API call outside the transaction, added PgBouncer, set statement and idle-in-transaction timeouts, and sized pools from the math.

6. **Q: How did you confirm?**
   A: Pending zero at the next peak; connections at ~40% of max; a load test with max replicas passed.

7. **Q: What if killing sessions didn't help — the pool filled again immediately?**
   A: The underlying hold time is still high: find the slow query/lock or the slow dependency, or the leak. Roll back the latest deploy, and reduce incoming load (shed non-critical traffic) while you look.

8. **Q: Why can scaling out make it worse?**
   A: Each new pod brings its own pool. The DB, not the app, is the bottleneck; more pods means more connections and more contention on the same DB.

9. **Q: What would you do differently?**
   A: Put a pooler in front of the DB from the start and link HPA limits to DB capacity in the design.

10. **Q: What if it happened at 10× traffic?**
    A: The math breaks immediately: you need a pooler, read replicas for read traffic, caching to cut queries, and load shedding so the DB isn't pushed past its limit.
