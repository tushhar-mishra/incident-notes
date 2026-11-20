# 31 · Slow Queries, Locks and Migrations

> **Level:** ★★★ advanced
>
> **You will learn:** how one slow query or one **lock** can take down a whole application, the **lock queue** problem (a harmless-looking `ALTER TABLE` blocking every read), **query plan flips** (the database suddenly picks a bad plan after statistics change), **missing indexes** after data growth, **long transactions** and vacuum, how to find the culprit with `pg_stat_activity`, `EXPLAIN` and `pg_stat_statements`, and how to run **schema migrations** safely (lock timeouts, online schema change, expand/contract).
>
> **You need:** topics 01–30. Helpful: [SD 16 · Database fundamentals](../../System-Design/notes/16-database-fundamentals.md), [SD 35 · Zero-downtime database changes](../../System-Design/notes/35-zero-downtime-database-changes.md).
>
> **Part 2 · Recurring failures · E. Data** · Previous: [30 · Connection-pool exhaustion](30-connection-pool-exhaustion.md) · Next: [32 · Replication, failover and split brain](32-replication-failover-split-brain.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The database is shared by every request. If **one** query becomes slow, or **one** session holds a lock, everyone else queues behind it, connection pools fill (topic 30), and the whole app goes down.

```
Session A: long-running SELECT on orders (holds a light lock)
Session B: ALTER TABLE orders ADD COLUMN ...  → needs an EXCLUSIVE lock → WAITS for A
Session C, D, E... (normal SELECTs on orders) → queue BEHIND B  ← everything stops
```

**Picture it.** A single-lane bridge. A slow tractor (long query) is crossing. A truck (migration) waits to cross the whole bridge alone. Every car behind the truck must wait too, even though cars could have shared the bridge with the tractor.

**The one thing to know.** For PostgreSQL migrations, **always set `lock_timeout`** (e.g. `SET lock_timeout = '3s'`) so the migration gives up instead of blocking everyone. And watch for **query plan flips**: the same query, no code change, suddenly 100× slower.

**Try it now:** `EXPLAIN ANALYZE SELECT ...` on a query you know. Find "Seq Scan" vs "Index Scan".

**You've got it if you can:** find a blocking session, explain a lock queue, and write a safe migration.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: [72 · Clerk 2026](72-clerk-2026.md), [73 · GitHub 2026 mini-cases](73-github-2026-mini-cases.md).

---

## Symptoms

| Where | What you see |
|---|---|
| App | Latency climbs on endpoints touching one table, then everything (pools fill); timeouts |
| DB metrics | Active sessions spike; many sessions `wait_event_type = Lock`; CPU high (bad plan) or low (lock wait) |
| DB logs | `canceling statement due to lock timeout`, `deadlock detected`, `canceling statement due to statement timeout` |
| Timing | Right after a migration started; after a big data load; after statistics changed (auto-ANALYZE) with **no** deploy |
| Pattern | One query's mean time jumps 10–1000× in `pg_stat_statements` |

---

## Analogy

The **single-lane bridge** (Core). A **plan flip** is like a GPS that suddenly sends every driver down a narrow farm track because its map data was updated with a wrong estimate.

---

## How it happens

### Lock queues

In PostgreSQL, many `ALTER TABLE` forms need an **ACCESS EXCLUSIVE** lock. Locks are granted **in order**: once the ALTER is waiting, later queries that conflict with it (including plain SELECTs) wait behind it. A migration that would take 10 ms can stop traffic for minutes if a long query was running.

### Query plan flips

The query **planner** picks a plan (which index, which join order) using **statistics** (row counts, value distributions) gathered by `ANALYZE` (run automatically by autovacuum). If statistics change — data grew, distribution shifted, or auto-ANALYZE sampled differently — the planner can pick a **much worse plan** for the same query. No code change, no deploy. Clerk's February 2026 outage (topic 72) was traced to this kind of plan flip after automatic ANALYZE.

### Other common causes

| Cause | Example |
|---|---|
| **Missing index after growth** | Query was fine at 10k rows, sequential scan at 50M rows |
| **Long transactions** | Hold locks; block vacuum → table/index bloat → everything slower |
| **Big batch/backfill** | Saturates I/O and connections at peak (GitHub May 2026 report: a migration plus peak traffic saturated connections, topic 73) |
| **N+1 queries in new code** | 1 request → 500 queries |
| **Deadlocks** | Two transactions lock rows in opposite order |

---

## How to find it

**First:** is the DB slow because of **locks** (waiting) or **work** (CPU/I/O)?

```sql
SELECT wait_event_type, wait_event, state, count(*)
FROM pg_stat_activity WHERE state <> 'idle'
GROUP BY 1,2,3 ORDER BY 4 DESC;
```
```
 wait_event_type | wait_event | state  | count
-----------------+------------+--------+-------
 Lock            | relation   | active |   212
 (null)          | (null)     | active |     4
```
- **212 sessions waiting on a relation lock** → lock queue.

Find the head of the queue:
```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, now() - query_start AS waiting, left(query, 70)
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0
ORDER BY waiting DESC LIMIT 5;

SELECT pid, now() - xact_start AS age, state, left(query, 70)
FROM pg_stat_activity WHERE pid = <blocker_pid>;
```
```
  pid  | blocked_by | waiting  | left
-------+------------+----------+-----------------------------------------
 40112 | {39877}    | 00:02:41 | ALTER TABLE orders ADD COLUMN gift_note text
 40150 | {40112}    | 00:02:30 | SELECT * FROM orders WHERE id = $1
```
- The ALTER (40112) waits for 39877; every SELECT waits for the ALTER.

**Second:** if it's **work**, which query got expensive?

```sql
SELECT left(query, 60), calls, round(mean_exec_time::numeric, 1) AS mean_ms,
       round(total_exec_time::numeric/1000) AS total_s
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;
```
Compare with yesterday (snapshots or your monitoring tool). A query whose `mean_ms` jumped from 2 to 900 → plan flip or missing index.

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;    -- ⚠️ ANALYZE really runs the query; use on a replica or with care
```
Look for: `Seq Scan` on a large table, `Rows Removed by Filter: 49,000,000`, nested loops with huge row estimates vs actual (`rows=1` estimated, `rows=2000000` actual → bad statistics).

**Third:** what changed? Migrations running, a backfill job, auto-ANALYZE timestamps:

```sql
SELECT relname, last_analyze, last_autoanalyze, n_live_tup FROM pg_stat_user_tables
ORDER BY last_autoanalyze DESC NULLS LAST LIMIT 5;
```

---

## How to fix it

### Immediate

- **Lock queue:** cancel the **waiting migration** (`SELECT pg_cancel_backend(40112);`) — traffic resumes at once. Or terminate the old blocker if it's safe.
- **Plan flip:** re-run `ANALYZE <table>;` (statistics may come back better), add the missing index **concurrently** (`CREATE INDEX CONCURRENTLY`), or rewrite/hint the query; roll back a code change if it introduced the query.
- **Backfill/batch:** pause it.
- Set a temporary `statement_timeout` for the app role to stop pile-ups.

### Permanent

- **Safe migrations:**
  ```sql
  SET lock_timeout = '3s';          -- give up rather than block everyone
  SET statement_timeout = '15min';
  ALTER TABLE orders ADD COLUMN gift_note text;   -- nullable, no default rewrite: fast
  ```
  Retry with backoff in the migration tool. Create indexes `CONCURRENTLY`. Use online schema change tools for MySQL (gh-ost, pt-online-schema-change). **Expand/contract** for renames and type changes.
- Run migrations **off-peak**, one at a time, watched.
- Throttled backfills (batches with sleeps; stop when replication lag or latency rises).
- Plan stability: monitor query latency per query fingerprint; increase statistics targets on skewed columns; consider extended statistics; test plans on production-size data.
- `idle_in_transaction_session_timeout`, statement timeouts per role.

---

## How to verify the fix

- No sessions waiting on locks; active sessions back to baseline.
- The offending query's mean time back to normal (`pg_stat_statements` reset or compared over a window).
- App latency/error rates normal; connection pools healthy.

---

## Prevention

- Alerts: sessions waiting on locks > N; longest transaction age; per-query latency regressions.
- Migration linting in CI (e.g. squawk for PostgreSQL) to flag dangerous statements.
- Production-size staging data for performance tests.
- Runbook: "how to find and cancel a blocker".

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a migration `ALTER TABLE` waited behind a long analytics query, and every app query queued behind the ALTER. Root cause: migrations ran without `lock_timeout` during business hours; analytics ran on the primary.

2. **Q: How did you detect it? Why not earlier?**
   A: Latency alert and pool exhaustion. No alert on lock waits existed.

3. **Q: What did you check first?**
   A: `pg_stat_activity` wait events, to tell lock waiting from expensive work; then `pg_blocking_pids` to find the head of the queue.

4. **Q: How did you stop impact quickly?**
   A: Cancelled the migration, which released the queue immediately.

5. **Q: Permanent fix?**
   A: Migration tool sets lock_timeout with retries; analytics moved to a replica; migration linting in CI.

6. **Q: How did you confirm?**
   A: Lock waits zero; latency normal; the migration later succeeded off-peak in milliseconds with lock_timeout set.

7. **Q: What if cancelling the migration didn't help?**
   A: Then the blocker is something else (another long transaction, a vacuum/DDL elsewhere) or the cause is expensive work, not locks. Re-check wait events and blocking pids; look at top queries by total time.

8. **Q: A query got 100× slower with no deploy. What happened?**
   A: Likely a plan change from new statistics or data growth. Compare EXPLAIN plans and estimates vs actuals; re-ANALYZE, add the right index concurrently, or adjust statistics.

9. **Q: What would you do differently?**
   A: Treat migrations as deploys: reviewed, linted, timed off-peak, with lock timeouts, and with a runbook to cancel.

10. **Q: What if it happened during a deployment at 10× traffic?**
    A: The lock queue fills 10× faster, so lock_timeout must be short and the migration must not run at peak. At that scale use online schema change tools and expand/contract patterns exclusively.
