# 32 · Replication Lag, Failover and Split Brain

> **Level:** ★★★ advanced
>
> **You will learn:** how database **replication** works (primary → replicas), what **replication lag** breaks (stale reads, "my order disappeared"), how **automatic failover** works and how it goes wrong (**promoting a lagging replica** = lost writes; **failing over across regions** = slow everything), what **split brain** is (two primaries accepting writes), how **quorum** and **fencing** prevent it, and the commands to check lag and roles on PostgreSQL and MySQL.
>
> **You need:** topics 01–31. Helpful: [SD 17 · Replication and failover](../../System-Design/notes/17-replication-and-failover.md), [SD 19 · Consistency, CAP, PACELC](../../System-Design/notes/19-consistency-cap-pacelc.md), [SD 26 · Distributed systems realities](../../System-Design/notes/26-distributed-systems-realities.md).
>
> **Part 2 · Recurring failures · E. Data** · Previous: [31 · Slow queries and locks](31-slow-queries-locks-and-migrations.md) · Next: [33 · ID and integer overflow](33-id-and-integer-overflow.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **primary** database accepts writes and streams them to **replicas**. Replicas are a little **behind** (lag). If the primary dies, a replica is **promoted**. Three things go wrong:

```
1. LAG          primary ──(changes)──► replica (5 s behind)  → users read stale data
2. BAD FAILOVER promote a replica that was 30 s behind      → 30 s of committed writes LOST
3. SPLIT BRAIN  network partition: old primary still alive + new primary promoted
                → BOTH accept writes → two diverging histories → manual reconciliation
```

**Picture it.** Two shop managers with a shared order book. Normally one writes and the other copies. If they lose contact and **both** start taking orders in their own copies, by morning you have two different order books and angry customers.

**The one thing to know.** Automatic failover needs **quorum** (a majority must agree who's primary) and **fencing** (the old primary must be stopped from writing). Without them, a short network blip can cause split brain — or a long, painful recovery, as in **GitHub's October 2018 outage** (topic 71).

**Try it now:** in your PostgreSQL lab, `SELECT pg_is_in_recovery();` on primary and replica.

**You've got it if you can:** check lag, explain why failover can lose data, and describe fencing.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Users | "I saved it but it's not there" (read-after-write on a lagging replica); data appears then disappears |
| Metrics | Replica lag (seconds or bytes) climbing; `pg_stat_replication` replay_lag high |
| Failover events | Orchestrator/Patroni/RDS event: "promoted replica X"; app errors during switch (read-only errors, connection resets) |
| Split brain | Two nodes report being primary; conflicting rows; duplicate IDs; replication broken afterwards |
| Cross-region failover | All writes suddenly slow (each write now crosses a continent) |

---

## Analogy

The **two shop managers** (Core). **Fencing** is taking the old manager's pen away before handing the book to the new one.

---

## How it happens

### Replication lag

Async replication: the primary commits, then sends changes. Lag grows when:
- Write volume spikes (bulk load, backfill, migration).
- Replica is under-sized or busy (heavy reads, long queries conflicting with replay).
- Network between them is slow or saturated.
- A long transaction or a large DDL replays slowly.

### Failover risks

| Risk | Why |
|---|---|
| **Data loss** | With **asynchronous** replication, the promoted replica may not have the last writes. **Synchronous** replication avoids this but adds write latency. |
| **Wrong candidate** | Failover tool picks a replica in another region (latency) or a lagging one |
| **Flapping** | Failover triggered by a short network blip, then again, and again |
| **Apps don't follow** | Apps keep connecting to the old primary (cached DNS, hard-coded IPs) and get read-only errors |
| **Split brain** | Old primary was only **unreachable**, not dead; it keeps accepting writes from clients that can still reach it |

### Quorum and fencing

- **Quorum:** a decision (e.g. "node B is now primary") needs a **majority** of voters (e.g. 2 of 3, 3 of 5). A minority side of a partition can't promote anyone. This is why consensus systems (etcd, Consul, ZooKeeper) use odd numbers of members.
- **Fencing:** guarantee the old primary can't write: revoke its storage access, shut it down via the cloud API (STONITH: "Shoot The Other Node In The Head"), or use **fencing tokens** (each new primary gets a higher number; storage rejects writes with an older number).
- **Leases:** the primary must keep renewing a lease in a consensus store; if it can't, it **demotes itself**.

---

## How to find it

**First:** who is primary right now, and is there only one?

```sql
-- PostgreSQL, on every node
SELECT pg_is_in_recovery();     -- false = primary, true = replica
```
```bash
patronictl -c /etc/patroni.yml list        # if using Patroni
```
```
+ Cluster: pg-main ----+---------+---------+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| pg1    | 10.0.1.10   | Leader  | running |  7 |           |
| pg2    | 10.0.2.10   | Replica | running |  7 |         0 |
| pg3    | 10.0.3.10   | Replica | running |  6 |       512 |
+--------+-------------+---------+---------+----+-----------+
```
- One leader → no split brain.
- `pg3` on **timeline (TL) 6** while others are on 7 → it didn't follow the last promotion; it needs re-syncing.

MySQL: `SELECT @@read_only, @@super_read_only;` and `SHOW REPLICA STATUS\G` (`Seconds_Behind_Source`, `Replica_IO_Running`, `Replica_SQL_Running`).

**Second:** how much lag?

```sql
-- On the primary
SELECT application_name, client_addr, state, write_lag, flush_lag, replay_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS bytes_behind
FROM pg_stat_replication;
-- On a replica
SELECT now() - pg_last_xact_replay_timestamp() AS replay_delay;
```
(Note: `replay_delay` on an idle primary looks large even with no lag, because no new transactions arrive.)

**Third:** was there a failover, and why? Failover tool logs (Patroni, Orchestrator, RDS events), network metrics between AZs/regions at the time, and the app's connection target (DNS/proxy).

---

## How to fix it

### Immediate

- **Lag causing stale reads:** route read-after-write traffic to the primary (session stickiness), pause the backfill causing lag, add replica capacity.
- **Bad failover to a far region:** fail back to a local candidate **once it's caught up and safe**, or move app traffic to the same region as the new primary.
- **Split brain:** **stop writes to one side immediately** (fence the old primary: shut it down or block its clients), pick the side with the authoritative data, then plan reconciliation of the writes made on the other side (export them, compare, replay by hand or by script). This can take hours — GitHub's 2018 recovery took about 24 hours largely because of this.
- Don't let automation flap: temporarily disable automatic failover while humans are in control.

### Permanent

- Failover with **quorum** (Patroni + etcd/Consul; managed services like RDS Multi-AZ/Aurora) and **fencing**.
- Rules for candidates: same region preferred; maximum allowed lag before promotion; synchronous replica for zero-data-loss needs.
- Failover **dampening**: require the primary to be unreachable for long enough; prevent repeated failovers.
- Apps connect through a stable endpoint (proxy/DNS with low TTL) and handle read-only errors by reconnecting.
- **Practise failover** regularly (game days), including "what if the network blips for 40 seconds?".

---

## How to verify the fix

- Exactly one primary; all replicas on the same timeline, streaming, lag near zero.
- Writes succeed; read-after-write consistent for users.
- Reconciliation complete and verified (row counts, checksums for affected tables) if split brain happened.

---

## Prevention

- Alerts: replica lag > N seconds; replication stopped; more than one writable node; failover events.
- Regular failover drills; documented RPO (Recovery Point Objective: how much data you can lose) and RTO (Recovery Time Objective: how fast you must be back).
- Topology review: where are primaries and candidates, and what happens on a cross-region failover?

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a 40-second network partition between data centres triggered automatic failover to a replica in another region; writes from the old primary that hadn't replicated were left behind, and app latency jumped because writes crossed regions. Root cause: failover tooling allowed cross-region promotion without considering topology or unreplicated writes.

2. **Q: How did you detect it? Why not earlier?**
   A: Latency alerts and failover events. The risk wasn't visible because failover had only been tested within a region.

3. **Q: What did you check first?**
   A: How many nodes claimed to be primary, and where — split brain changes every priority (stop writes first).

4. **Q: How did you stop impact quickly?**
   A: Paused automated topology changes, blocked writes on the stale side, and degraded non-essential write features to protect data.

5. **Q: Permanent fix?**
   A: Topology-aware failover rules, quorum and fencing, and failover drills including network partitions.

6. **Q: How did you confirm?**
   A: Single primary, replication healthy everywhere, data reconciliation verified, latency normal.

7. **Q: What if failing back failed?**
   A: Stay on the new primary and move app traffic close to it, rather than risk another switch. Rebuild the old primary as a replica from a fresh base backup, then fail back in a planned window.

8. **Q: Synchronous vs asynchronous replication: when to use each?**
   A: Synchronous guarantees no committed write is lost on failover, at the cost of write latency and availability (if the sync replica is down, writes may stall). Asynchronous is faster but accepts losing recent writes. Many teams use one synchronous replica nearby plus async replicas further away.

9. **Q: What would you do differently?**
   A: Test failover under realistic partitions, including short ones, and make sure automation can't pick a candidate that breaks latency or loses data.

10. **Q: What if it happened during a deployment?**
    A: Freeze deploys immediately; a deploy plus a database topology change makes it impossible to tell causes apart, and new code might write in a way the reconciliation must handle too.
