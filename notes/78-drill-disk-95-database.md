# 78 · Drill: Disk at 95% on the Database Host

> **Level:** ★ foundation
>
> **You will practise:** freeing disk space **safely** on a host where the wrong command can corrupt a database: finding what grew, telling **logs** from **WAL** from **data**, truncating instead of deleting, growing a cloud volume online, and stopping the growth.
>
> **You need:** topics 13, 31, 34, 76.
>
> **Part 4 · On-call drills** · Previous: [77 · 5xx after a deploy](77-drill-5xx-after-deploy.md) · Next: [79 · Latency up, nothing deployed](79-drill-latency-no-deploy.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **03:20 — PAGE:** `NodeFilesystemAlmostOutOfSpace — pg-prod-1 /var/lib/postgresql 95% used, will fill in ~40 min`

**The idea.** You have about 40 minutes before PostgreSQL can't write (and may shut down). **Buy time safely** (grow the volume, remove non-database files), then **find and stop the growth**.

```
1. Which filesystem, how fast is it growing?          df -h, the graph
2. What's big: logs? WAL (pg_wal)? data? dumps?        du, lsof +L1
3. Safe space now: grow volume / truncate logs / delete old dumps — NEVER touch pg_wal by hand
4. Stop the cause: runaway log level, stuck replication slot, failed archiving, backup to wrong disk
```

**The one thing to know.** **Never delete files inside the PostgreSQL data directory or `pg_wal` by hand.** Find why WAL can't be recycled instead.

> 📖 Read only **Symptoms**, then write your first three checks.

---

## Symptoms

- Disk prediction alert; disk graph is a **steep ramp**, not a slow line.
- Possibly: application latency on writes; PostgreSQL log warnings about archiving failures.
- If it fills: `PANIC: could not write to file "pg_wal/...": No space left on device`, database stops accepting writes.

## Analogy

The kitchen bin is overflowing during dinner service. You take out the recycling (safe) and get a bigger bin (grow the volume) — you don't throw away the food in the fridge (database files) to make space.

## How it happens

Common causes on database hosts (topic 13):
- **WAL can't be removed**: an abandoned **replication slot** (a replica that was deleted but its slot remains) or **failing WAL archiving** (`archive_command` failing) keeps WAL forever.
- **Logs**: `log_statement = 'all'` left on after debugging.
- **Backups/dumps** written to the database disk.
- Real data growth (a big import).

### Lab setup (break it)

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=pw --tmpfs /var/lib/postgresql/data:size=300m postgres:16
sleep 5
# Create a replication slot that nobody consumes → WAL retained
docker exec pg psql -U postgres -c "SELECT pg_create_physical_replication_slot('old_replica', true);"
# Generate writes → WAL grows
docker exec pg psql -U postgres -c "CREATE TABLE t AS SELECT g, repeat('x',1000) s FROM generate_series(1,100000) g;"
docker exec pg psql -U postgres -c "DO \$\$BEGIN FOR i IN 1..20 LOOP UPDATE t SET s = s; END LOOP; END\$\$;"
```

## How to find it

**First — which filesystem, how fast:**

```bash
df -h /var/lib/postgresql
```
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme1n1    200G  190G   10G  95% /var/lib/postgresql
```

**Second — what's using it:**

```bash
sudo du -xh --max-depth=2 /var/lib/postgresql | sort -h | tail -5
```
```
2.1G    /var/lib/postgresql/16/main/log
41G     /var/lib/postgresql/16/main/base
146G    /var/lib/postgresql/16/main/pg_wal      ← WAL is the problem, not data
```

**Third — why WAL isn't recycled:**

```sql
SELECT slot_name, slot_type, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```
```
  slot_name  | slot_type | active | retained
-------------+-----------+--------+----------
 old_replica | physical  | f      | 145 GB      ← inactive slot holding 145 GB of WAL
```
```sql
SELECT archived_count, failed_count, last_failed_wal, last_failed_time FROM pg_stat_archiver;
```
(`failed_count` growing = archiving broken, another reason WAL is kept.)

Also check `sudo lsof +L1` for deleted-but-open log files (topic 13).

## How to fix it

### Immediate (safe order)

1. **Buy time:** grow the cloud volume online (EBS `modify-volume`, then `xfs_growfs` / `resize2fs`) — no downtime.
2. **Remove non-database files:** old rotated logs, stray dumps (`find ... -mtime +7 -delete`), truncate an active huge log (`truncate -s 0`).
3. **Fix the WAL cause:**
   - Abandoned slot, **after confirming no replica uses it** (check with the team/inventory):
     ```sql
     SELECT pg_drop_replication_slot('old_replica');
     CHECKPOINT;
     ```
     PostgreSQL then recycles WAL.
   - Broken archiving: fix `archive_command` (credentials, bucket, network).
4. ⚠️ Don't `rm` anything in `pg_wal` or `base`. Don't restart PostgreSQL "to see if it helps".

### Permanent

- `max_slot_wal_keep_size` (PostgreSQL 13+) so an abandoned slot can't fill the disk.
- Alerts: replication slot retained WAL, inactive slots, archiver failures, disk prediction.
- Separate volumes for data, WAL and logs; backups never on the DB volume.
- Runbook for "disk filling on DB host".

## How to verify the fix

- `df -h` falling or stable; `pg_wal` size back to normal after a checkpoint.
- `pg_replication_slots` shows no inactive slots retaining large WAL; `pg_stat_archiver.failed_count` not increasing.
- Disk prediction alert cleared; writes healthy.

## Prevention

`max_slot_wal_keep_size`, slot and archiver monitoring, volume layout, log retention, capacity trend reviews.

## Follow-up questions

1. **Q: Root cause, not trigger?**
   A: Trigger: disk 95%. Root cause: a replication slot left behind when a replica was removed, retaining WAL indefinitely, with no slot monitoring or WAL retention limit.
2. **Q: How did you detect it? Why not earlier?**
   A: Disk prediction alert with ~40 minutes left. Earlier: an alert on inactive slots or retained WAL would have fired days before.
3. **Q: What did you check first, and why?**
   A: Which directory was growing; on a DB host, WAL vs data vs logs need completely different fixes.
4. **Q: How did you stop impact quickly?**
   A: Grew the volume online to buy time, then dropped the confirmed-abandoned slot.
5. **Q: Permanent fix?**
   A: `max_slot_wal_keep_size`, slot/archiver alerts, separate volumes, decommissioning checklist for replicas.
6. **Q: How did you confirm?**
   A: WAL size dropped after checkpoint; disk trend flat; alerts cleared.
7. **Q: What if dropping the slot wasn't possible (a replica still needs it)?**
   A: Grow the disk more and fix the lagging replica (or rebuild it from a fresh base backup later); never drop a slot a live replica uses.
8. **Q: How would you prevent it?**
   A: WAL retention limits and monitoring, plus replica decommission runbooks that drop slots.
9. **Q: What would you do differently?**
   A: Grow the volume first, immediately, before investigating — it's safe and buys hours.
10. **Q: What if you had no access to grow the volume and 5 minutes left?**
    A: Free the largest safe non-database files (old logs, dumps), and if WAL archiving is the issue and archives are confirmed elsewhere, fix archiving. As a last resort, stop non-critical writers to slow WAL growth. Never delete WAL by hand.
