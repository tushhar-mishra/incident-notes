# 33 · ID and Integer Overflow

> **Level:** ★★ intermediate
>
> **You will learn:** why **32-bit integers** run out (2,147,483,647 for signed `INT`), how **auto-increment IDs** and **sequences** hit the limit, other "counters that run out" (PostgreSQL **transaction ID wraparound**, Linux **PID** and **ephemeral port** ranges, **2038** timestamps), what breaks when they do, how to **see it coming** with simple queries, and how to migrate a primary key from `INT` to `BIGINT` without downtime.
>
> **You need:** topics 01–32. Helpful: [SD 18 · Partitioning and sharding](../../System-Design/notes/18-partitioning-and-sharding.md), [SD 35 · Zero-downtime DB changes](../../System-Design/notes/35-zero-downtime-database-changes.md).
>
> **Part 2 · Recurring failures · E. Data** · Previous: [32 · Replication and split brain](32-replication-failover-split-brain.md) · Next: [34 · Backups that don't restore](34-backups-that-dont-restore.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every number type has a maximum. A 32-bit signed integer stops at **2,147,483,647** (about 2.1 billion). A busy table can use that up in a few years. On the day the next ID doesn't fit, **every insert fails**, and nothing else is wrong.

```
INT (32-bit signed)     max 2,147,483,647          ≈ 2.1 billion
INT UNSIGNED            max 4,294,967,295          ≈ 4.3 billion
BIGINT (64-bit signed)  max 9,223,372,036,854,775,807 ≈ 9.2 quintillion  (practically never)
```

GitHub's **May 2026** availability report (topic 73) describes an incident caused by a **32-bit integer key reaching its maximum value** in a lookup table used when creating pull request threads.

**Picture it.** A ticket machine at a deli with 3-digit numbers. After ticket 999, it can't print 1000. The queue stops, even though the deli is open and staff are ready.

**The one thing to know.** It is **100% predictable**. One query tells you how close each table is. Check it monthly and alert at 50–70%.

**Try it now:** in any PostgreSQL lab, find all `integer` primary keys and their sequences' current values (query below).

**You've got it if you can:** compute "days until exhaustion" for a table and plan a safe BIGINT migration.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| PostgreSQL | `ERROR: nextval: reached maximum value of sequence "orders_id_seq" (2147483647)` or `integer out of range` |
| MySQL | `ERROR 1062 (23000): Duplicate entry '2147483647' for key 'PRIMARY'` (auto-increment stuck at max) or `ERROR 167: Out of range value` |
| App | All **inserts** into one table fail; reads work fine; one feature completely broken ("can't create orders/comments") |
| Elsewhere | Another system storing the ID in a 32-bit field (app code, a queue message schema, an analytics column) breaks **even if** the DB column is BIGINT |
| PostgreSQL XID | `database is not accepting commands to avoid wraparound data loss` (transaction ID wraparound protection) |

---

## Analogy

The **deli ticket machine** (Core).

---

## How it happens

| Counter | Limit | How it runs out |
|---|---|---|
| **Auto-increment / sequence IDs** (`INT`/`SERIAL`) | ~2.1 billion | Growth; **gaps** (failed inserts and rollbacks still consume sequence values; some bulk inserts reserve ranges) make it faster than row count suggests |
| **IDs stored in 32-bit fields elsewhere** | ~2.1 billion | DB moved to BIGINT, but a client library, protobuf `int32`, or a column in another table still uses 32-bit |
| **PostgreSQL transaction IDs (XID)** | 32-bit, wraps around | If (auto)vacuum can't freeze old rows (long transactions, blocked vacuum), the database approaches wraparound and eventually refuses writes to protect data |
| **Linux PIDs** | `kernel.pid_max` (default 32768 on many systems, up to ~4 million) | Fork bombs or thread leaks → `fork: Resource temporarily unavailable` (and Kubernetes PIDPressure) |
| **Ephemeral ports** | `net.ipv4.ip_local_port_range` (often ~28,000 ports) | Many outgoing connections to the same destination, TIME_WAIT → `cannot assign requested address` |
| **Unix time in 32-bit signed** | 19 January 2038, 03:14:07 UTC | Old systems/file formats storing time as 32-bit seconds |
| **Counters in your own code** | Whatever type you chose | A retry counter, a version number, a byte counter that wraps |

---

## How to find it

**First:** which table/sequence is at the limit?

```sql
-- PostgreSQL: how full is every sequence?
SELECT schemaname, sequencename, data_type, last_value, max_value,
       round(100.0 * last_value / max_value, 1) AS pct_used
FROM pg_sequences
WHERE last_value IS NOT NULL
ORDER BY pct_used DESC LIMIT 10;
```
```
 schemaname | sequencename    | data_type | last_value | max_value  | pct_used
------------+-----------------+-----------+------------+------------+---------
 public     | comments_id_seq | integer   | 2147483647 | 2147483647 |   100.0
 public     | events_id_seq   | integer   | 1610612736 | 2147483647 |    75.0
 public     | orders_id_seq   | bigint    | 98000000   | 9.2e18     |     0.0
```
- `comments` is **exhausted**; `events` is at 75% → next.

Note: a `bigint` sequence feeding an `integer` column still fails at the column limit. Check column types too:
```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE data_type = 'integer' AND column_name ~ '(^id$|_id$)';
```

MySQL:
```sql
SELECT table_schema, table_name, auto_increment
FROM information_schema.tables
WHERE auto_increment IS NOT NULL ORDER BY auto_increment DESC LIMIT 10;
```

**Second:** how fast is it growing? (days to exhaustion)

```
days_left = (max_value - last_value) / ids_used_per_day
```
Measure `ids_used_per_day` from two readings a day apart (not from row counts, because of gaps).

**Third:** PostgreSQL XID age:
```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
```
Values approaching ~2 billion are dangerous; autovacuum normally keeps this far lower (it starts aggressive "anti-wraparound" vacuums at `autovacuum_freeze_max_age`, default 200 million).

---

## How to fix it

### Immediate

- **Exhausted INT key:** there's no instant fix that keeps inserting into the same INT column. Options, fastest first:
  - If the column is `INT` but the app can tolerate it: in some designs, switch the sequence to **negative** numbers (`ALTER SEQUENCE ... MINVALUE -2147483648 START ... INCREMENT -1`) to buy time — only if no code assumes IDs are positive. Risky; test first.
  - Change the column to `BIGINT` — on a big table, a direct `ALTER COLUMN TYPE bigint` **rewrites the table** under an exclusive lock (can take hours of downtime).
  - Disable the feature that inserts, communicate, and do the proper migration.
- **XID wraparound:** run `VACUUM (FREEZE)` on the oldest tables; kill long transactions that block vacuum. If the DB already refuses commands, follow the PostgreSQL docs procedure (single-user mode vacuum in older versions).
- **PIDs/ports:** raise `pid_max` / widen `ip_local_port_range`, enable connection reuse; kill the leaking process.

### Permanent: the online INT → BIGINT migration

```
1. Add new column id_new BIGINT (nullable, no default → fast)
2. Trigger: on insert/update, copy id → id_new
3. Backfill old rows in small batches (throttled, off-peak)
4. Build a unique index CONCURRENTLY on id_new
5. Short maintenance step: swap columns/constraints (primary key on id_new), update sequence type, rename
6. Fix every foreign key and every client/schema that stores the ID as 32-bit
```
(Tools like gh-ost / pt-online-schema-change do the equivalent for MySQL.)

- **Default to BIGINT** (or UUIDs) for new tables' primary keys.
- Check the **whole path**: app code types, APIs, message schemas, data warehouse columns.

---

## How to verify the fix

- Inserts succeed; `pct_used` for the migrated sequence ~0%.
- No 32-bit fields remain in the path (schema review + a test inserting an ID above 2^31).
- XID age back to normal ranges.

---

## Prevention

- Monthly (or continuous) check of sequence usage; alert at 50% (ticket) and 75% (page-worthy plan).
- Lint new migrations: reject `INT`/`SERIAL` primary keys.
- Test fixtures that use IDs above 2^31 so 32-bit assumptions fail in CI.
- Monitor XID age, PID usage, ephemeral port usage.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: The trigger was the sequence hitting 2,147,483,647. The root cause was choosing a 32-bit primary key years ago and never monitoring its growth; gaps from failed inserts made it arrive earlier than row counts suggested.

2. **Q: How did you detect it? Why not earlier?**
   A: Insert errors in one feature. It was 100% predictable, but nobody was checking sequence usage.

3. **Q: What did you check first?**
   A: The exact error, then `pg_sequences` for that table, to confirm it's the limit (not a constraint or a bug).

4. **Q: How did you stop impact quickly?**
   A: Disabled the feature with a clear message, and (after checking no code needed positive IDs) bought time with a negative-range sequence while running the online migration. If negatives weren't safe, the feature stays off until the migration finishes.

5. **Q: Permanent fix?**
   A: Online migration to BIGINT, fixing every downstream 32-bit field, and a lint rule for new tables.

6. **Q: How did you confirm?**
   A: Inserts working, an end-to-end test with IDs above 2^31 passing in every consumer.

7. **Q: What if the migration itself caused problems?**
   A: The backfill could saturate I/O or replication lag; throttle it, run off-peak, and pause on lag. The swap step needs a short lock: use lock_timeout and retry.

8. **Q: How would you prevent it?**
   A: Alert on sequence usage, BIGINT by default, and CI tests with large IDs.

9. **Q: What would you do differently?**
   A: Inventory all "counters that run out" (sequences, XIDs, PIDs, ports, quotas) and put them on one capacity dashboard.

10. **Q: What if it had happened on the main orders table at peak?**
    A: It's a SEV1: all new orders fail. The fastest safe option depends on code: a negative sequence if safe; otherwise a planned emergency maintenance window for the column change. That's why you migrate at 50%, not 100%.
