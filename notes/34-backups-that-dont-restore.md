# 34 · Backups That Don't Restore

> **Level:** ★★ intermediate
>
> **You will learn:** why "**we have backups**" means nothing until you've **restored** one, the many ways backups silently fail (job errors nobody reads, empty files, wrong version, missing keys, deleted together with production), **RPO** and **RTO**, the **3-2-1** rule and **immutable** backups (ransomware and "delete everything" scripts), **point-in-time recovery**, how to **verify** a backup automatically, and how to run a calm restore under pressure.
>
> **You need:** topics 01–33. Helpful: [SD 41 · Backups and disaster recovery](../../System-Design/notes/41-backups-and-disaster-recovery.md), [SD 52 · Database DR case](../../System-Design/notes/52-case-database-dr.md), [Linux 068 · rsync and backups](../../linux-mastery-rhel/notes/068-rsync-and-backups.md), [K8s 77 · etcd backup/restore](../../CKA-Certification-Course-2025/notes/77-etcd-backup-restore-and-dr.md), [Jenkins 68 · Backup and restore](../../Jenkins/notes/68-backup-restore-upgrades.md), [Terraform 42 · State recovery](../../Terraform/notes/42-state-recovery.md).
>
> **Part 2 · Recurring failures · E. Data** · Previous: [33 · ID and integer overflow](33-id-and-integer-overflow.md) · Next: [35 · Quotas and rate limits](35-quotas-and-rate-limits.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A backup is only useful if you can **restore** it, **fast enough**, to a **recent enough** point. Most teams discover their backups don't work during the incident that needs them. GitLab, January 2017 (topic 46): several backup methods turned out not to be working when they were needed.

```
RPO (Recovery Point Objective) = how much data can we lose?   e.g. ≤ 5 minutes of writes
RTO (Recovery Time Objective)  = how long can we be down?     e.g. ≤ 1 hour to restore

Backup taken ──► stored safely (off-site, immutable) ──► RESTORE TESTED regularly ──► measured time
```

**Picture it.** A spare tyre in your car boot. Having one doesn't help if it's flat, the wrong size, or the jack is missing. You only know if you've actually tried changing a tyre.

**The one thing to know.** **Test restores automatically and regularly**, and measure how long they take. An untested backup is a hope, not a backup.

**Try it now:** pick one database. When was its last successful restore test, and how long did it take?

**You've got it if you can:** define RPO/RTO for a service, and describe an automated restore test.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

(This failure is invisible until you need it. The "symptoms" appear during another incident.)

| Where | What you discover |
|---|---|
| Backup storage | Files are 0 bytes, or much smaller than expected; last successful backup weeks ago |
| Backup job | Has been failing silently (errors emailed to a list nobody reads; cron output to /dev/null) |
| Restore attempt | Wrong DB version (`pg_restore: unsupported version`), missing encryption key, corrupt archive, missing WAL segments for PITR |
| Timing | Restore takes 18 hours, but RTO is 1 hour (network/disk throughput never measured) |
| Scope | Backups were in the same account/region and were deleted/encrypted along with production (ransomware, bad script, compromised credentials) |
| Coverage | The database was backed up, but not the config, secrets, object storage, or the Kubernetes manifests needed to run it |

---

## Analogy

The **spare tyre** (Core).

---

## How it happens

| Failure | Example |
|---|---|
| **Silent job failure** | Backup script exits 0 even when `pg_dump` failed (missing `set -e`/`pipefail`) ([Linux 045 · Robust scripts](../../linux-mastery-rhel/notes/045-robust-scripts.md)) |
| **Monitoring the wrong thing** | Alert on "job ran", not on "backup is valid and recent" |
| **Version mismatch** | Backup tool version older than the server; restore needs a matching version |
| **Missing pieces** | Encryption keys stored only inside the system being backed up; WAL archive gaps; secrets not included |
| **Same blast radius** | Backups in the same account, region or storage that the incident destroyed |
| **Retention too short** | Corruption started 10 days ago; you keep only 7 days |
| **Restore never practised** | Nobody knows the steps; runbook outdated |
| **Too slow** | Multi-terabyte restore over limited bandwidth |

### Kinds of backup

| Type | What | RPO |
|---|---|---|
| **Logical dump** (`pg_dump`, `mysqldump`) | SQL/archive of data | Time of last dump (e.g. 24 h) |
| **Physical/base backup + WAL/binlog archive** | Copy of files + continuous log | Minutes or seconds (**PITR**, Point-In-Time Recovery) |
| **Snapshots** (EBS, RDS) | Block-level copy | Snapshot frequency; RDS automated backups support PITR |
| **Replicas** | Live copy | Not a backup! A `DROP TABLE` replicates instantly |

### 3-2-1 and immutability

- **3** copies of data, on **2** different media/types, **1** off-site (another account/region/provider).
- **Immutable** backups (S3 Object Lock, AWS Backup Vault Lock, write-once storage) can't be deleted by an attacker or a bad script for a retention period.
- Separate **credentials**: the role that runs production can't delete backups.

---

## How to find it

(Before the incident: audit. During the incident: find the newest **good** restore point.)

**First:** what is the newest backup, and is it plausible?

```bash
aws s3 ls s3://acme-db-backups/orders/ --recursive --human-readable | sort | tail -5
```
```
2026-11-17 02:00:12    4.1 GiB orders/2026-11-17.dump
2026-11-18 02:00:09    4.2 GiB orders/2026-11-18.dump
2026-11-19 02:00:03    0 Bytes orders/2026-11-19.dump      ← broken
2026-11-20 02:00:05    0 Bytes orders/2026-11-20.dump      ← broken
```
- Last two are empty → the newest usable is 11-18 (RPO now ~2 days unless WAL/PITR exists).

```bash
aws rds describe-db-instance-automated-backups --db-instance-identifier orders-prod \
  --query 'DBInstanceAutomatedBackups[].RestoreWindow'
aws backup list-recovery-points-by-backup-vault --backup-vault-name prod-vault --max-results 5
```

**Second:** can it be read?

```bash
pg_restore --list orders-2026-11-18.dump | head      # does the archive's table of contents load?
```

**Third:** restore into an **isolated** environment first (never over production while unsure), and check data:

```bash
createdb orders_restore_test
pg_restore -d orders_restore_test -j 8 orders-2026-11-18.dump
psql -d orders_restore_test -c "SELECT max(created_at), count(*) FROM orders;"
```

---

## How to fix it

### Immediate (during a data-loss incident)

1. **Stop the damage:** stop writes or the process causing loss; preserve what's left (snapshot the current broken state too — you may need it).
2. **Find the newest good restore point** (PITR timestamp just **before** the bad event).
3. **Restore to a new instance**, verify, then switch the app (or copy back only the lost data).
4. Communicate honestly about any data that can't be recovered.

```bash
# RDS point-in-time restore to a NEW instance
aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier orders-prod \
  --target-db-instance-identifier orders-pitr-0917 --restore-time 2026-11-20T09:17:00Z
```

### Permanent

- Backups with `set -euo pipefail`, checksums, and **size/row-count sanity checks**.
- **Automated restore tests** (daily/weekly): restore into a scratch environment, run checks, record duration, alert on failure.
- PITR for important databases; retention long enough to catch slow corruption.
- **Off-site + immutable** copies; separate backup account and credentials.
- Back up **everything** needed to run: data, config, secrets (in a secrets manager with its own backup), IaC state, manifests.
- Restore **runbook** practised in game days.

---

## How to verify the fix

- Restore test job succeeds on schedule; restored data passes checks (row counts, latest timestamp within RPO).
- Measured restore time ≤ RTO.
- Backups can't be deleted with production credentials (test it).

---

## Prevention

- Alerts on: no **successful verified** backup in N hours; backup size deviates > X% from usual; restore test failure.
- Quarterly DR game day (topic 90).
- RPO/RTO written down per service and reviewed with the business.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: the backup script piped `pg_dump` into `gzip` without `pipefail`, so when `pg_dump` failed (version mismatch after an upgrade), the script still exited 0 and uploaded an empty file. Root cause: no verification of backup content and no restore tests.

2. **Q: How did you detect it, and why not earlier?**
   A: Only during a data-loss incident. We monitored "job succeeded", not "backup restorable".

3. **Q: What did you check first?**
   A: The newest backup's size and readability, because I need the newest **good** restore point before deciding the recovery plan.

4. **Q: How did you stop impact?**
   A: Stopped the process deleting data, snapshotted the current state, and restored PITR to a new instance to recover the missing rows.

5. **Q: Permanent fix?**
   A: Robust scripts, size checks, automated weekly restore tests with timing, and immutable off-site copies.

6. **Q: How did you confirm?**
   A: A restore test passing every week, with restore time under RTO.

7. **Q: What if no backup was usable?**
   A: Look for every other copy: replicas (if they didn't replicate the damage yet), delayed replicas, data warehouses, event logs/queues, caches, application logs, third-party systems (payment provider records), even customer exports. Rebuild what you can and be transparent.

8. **Q: Is a replica a backup?**
   A: No. It protects against hardware failure, not against bad writes or deletes, which replicate immediately. Delayed replicas help a little; real backups and PITR are needed.

9. **Q: What would you do differently?**
   A: Measure backups by successful restores, not by successful jobs.

10. **Q: What if ransomware encrypted production and the attacker had admin credentials?**
    A: Only backups the attacker couldn't delete survive: immutable/object-locked copies in a separate account with separate credentials. Restore into a clean environment after the security team confirms the attacker is out.
