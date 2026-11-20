# 13 · Disk Full

> **Level:** ★ foundation
>
> **You will learn:** why "disk full" is one of the most common incidents anywhere, the **three different ways** a disk is "full" (**blocks**, **inodes**, **deleted-but-still-open files**), what breaks when it happens (databases, logs, containers, Kubernetes evictions), the exact **commands** to find what's using the space, how to free space **safely** (and the commands that make it worse), and how to make sure it never surprises you again.
>
> **You need:** Part 1. Helpful: [Linux 047 · Filesystems](../../linux-mastery-rhel/notes/047-filesystems.md), [Linux 008 · Inodes and links](../../linux-mastery-rhel/notes/008-inodes-and-links.md), [Docker 48 · Storage and disk problems](../../Docker/notes/48-storage-and-disk-problems.md).
>
> **Part 2 · Recurring failures · A. Linux host resources** · Previous: [12 · The root-cause map](12-the-root-cause-map.md) · Next: [14 · Too many open files](14-too-many-open-files.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A disk can be "full" in three ways:

```
1. BLOCKS full   df -h shows 100%            → big files (logs, dumps, images, data)
2. INODES full   df -i shows 100%            → millions of tiny files (sessions, cache, mail queue)
3. HIDDEN full   df says full, du says not   → a deleted file still held open by a process
```

**Picture it.** A library. Blocks = shelf space. Inodes = index cards (one per book). You can run out of shelf space (big books), or out of index cards (millions of tiny pamphlets) while shelves are half empty. A "deleted but open" file is a book removed from the catalogue but still sitting on the shelf because someone is reading it.

**The one thing to know.** **Never `rm` a log file that a process is still writing.** The space isn't freed until the process closes it. Use `truncate -s 0 file` (or restart the process) instead.

**Try it now:** `df -h; df -i` on any machine. Which filesystem is fullest?

**You've got it if you can:** explain why `df` and `du` can disagree, and fix it without a reboot.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Alerts | `NodeFilesystemAlmostOutOfSpace`, "disk will fill in 4 hours", `DiskPressure` on a Kubernetes node |
| Application logs | `No space left on device` (errno **ENOSPC**), `write error`, `could not extend file` |
| Databases | PostgreSQL: `could not write to file "pg_wal/...": No space left on device` → database may stop accepting writes or shut down. MySQL: `The table is full` / `Disk is full writing`. |
| Docker | `docker pull` / `docker build` fail: `no space left on device` |
| Kubernetes | Pods `Evicted` with reason `The node was low on resource: ephemeral-storage`; node condition `DiskPressure=True`; new pods can't schedule on the node |
| Users | Uploads fail, 500 errors, the site goes read-only, logins fail (session files can't be written) |
| Shell | Tab completion fails (`bash: cannot create temp file for here-document: No space left on device`) |

---

## Analogy

The **library** (Core). Also a **kitchen bin**: you can fill it with one big box (blocks), with a thousand sweet wrappers (inodes), or the bin looks empty but the bag is still full because someone is still holding the lid down (deleted-but-open).

---

## How it happens

### 1. Blocks full (the common case)

Something writes more data than expected:
- **Logs** with no rotation, or debug logging left on ([Linux 038 · logrotate](../../linux-mastery-rhel/notes/038-logging-journald-rsyslog-logrotate.md)).
- **Docker** container logs (`json-file` driver with no `max-size`), old images, build cache ([Docker 13](../../Docker/notes/13-cleanup-and-disk-usage.md), [Docker 33](../../Docker/notes/33-logging.md)).
- **Database** growth, WAL (Write-Ahead Log) files that can't be archived or removed because a **replication slot** or archiving is stuck.
- **Core dumps**, heap dumps, temp files, backups written to the same disk.
- **Jenkins** workspaces and build history ([Jenkins 22 · Disk hygiene](../../Jenkins/notes/22-scheduled-jobs-and-disk-hygiene.md)).

### 2. Inodes full

Each file uses one **inode** (index node: the metadata record — owner, permissions, where the data is). ext4 creates a fixed number of inodes when the filesystem is made; you can run out with space still free. (XFS allocates inodes dynamically, so this is rarer there, but still possible.)

Typical causes: PHP session files, mail queues, cache directories with millions of tiny files, a bug creating one file per request.

### 3. Deleted-but-open files

```
Process (nginx) has app.log open ──► inode 1234 (5 GB of data)
You run: rm app.log
  ▼
Directory entry removed (du can't see it any more)
But the process still has the file open → inode 1234 and its 5 GB stay allocated
df still counts the 5 GB as used.
```

The data is freed only when **the last open file descriptor is closed**.

### Reserved blocks

ext4 reserves 5% of blocks for root by default. So a normal user process may get `No space left on device` while `df` shows ~95% used. Root processes can keep writing a little longer.

---

## How to find it

**First:** which filesystem, and what kind of full?

```bash
df -hT
```
```
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/nvme0n1p1 xfs    50G   20G   30G  40% /
/dev/nvme1n1   ext4  200G  200G     0 100% /var/lib/postgresql
tmpfs          tmpfs 3.9G  1.2M  3.9G   1% /run
```
- `/var/lib/postgresql` is at **100%** → the database disk. This is urgent (database may stop).
- `Type` tells you ext4 (fixed inodes, 5% reserved) vs xfs.

```bash
df -i
```
```
Filesystem      Inodes   IUsed  IFree IUse% Mounted on
/dev/nvme0n1p1 26214400 26214400   0  100% /
```
- `IUse% 100%` while `df -h` showed 40% → **inode exhaustion**, not space.

**Second:** what's using the space?

```bash
sudo du -xh --max-depth=1 /var | sort -h | tail -5
```
```
1.2G    /var/cache
3.4G    /var/lib
41G     /var/log
46G     /var
```
- `-x` = stay on this filesystem (don't count other mounts).
- `/var/log` is the culprit. Drill down: `sudo du -xh --max-depth=1 /var/log | sort -h | tail`.

```bash
sudo find / -xdev -type f -size +1G -printf '%s %p\n' 2>/dev/null | sort -n | tail
```
Lists the biggest single files.

For inodes, find directories with the most files:
```bash
sudo find / -xdev -type f | cut -d/ -f2-3 | sort | uniq -c | sort -n | tail
```
```
  12043 var/lib
3810022 var/spool    ← millions of files
```

**Third:** do `df` and `du` disagree? Look for deleted-but-open files:

```bash
sudo lsof -nP +L1 | head
```
```
COMMAND   PID  USER  FD   TYPE DEVICE   SIZE/OFF NLINK    NODE NAME
java     4211  app   12w  REG  259,1 21474836480     0  393219 /var/log/app/app.log (deleted)
```
- `NLINK 0` and `(deleted)` → the file is deleted but `java` (PID 4211) still has it open on file descriptor 12, holding **20 GB**.

**Containers and Kubernetes:**

```bash
docker system df            # images, containers, volumes, build cache sizes
sudo du -sh /var/lib/docker/containers/*/*-json.log | sort -h | tail -3
kubectl describe node <node> | grep -A5 Conditions    # DiskPressure?
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted
```

---

## How to fix it

### Immediate (restore service)

| Situation | Safe action | ⚠️ Avoid |
|---|---|---|
| Big log file still being written | `sudo truncate -s 0 /var/log/app/app.log` (frees space, process keeps writing) | `rm` (space not freed while open) |
| Deleted-but-open file | Truncate through `/proc`: `sudo truncate -s 0 /proc/4211/fd/12`, or restart the process gracefully | Rebooting the database server in a panic |
| Old rotated logs | `sudo find /var/log/app -name '*.gz' -mtime +7 -delete` | `rm -rf /var/log/*` (deletes logs you need for this incident) |
| journald | `sudo journalctl --vacuum-size=500M` | — |
| Docker | `docker image prune -a` (unused images), `docker builder prune` | `docker system prune -a --volumes` on a host with data volumes (deletes data) |
| PostgreSQL WAL full | Find why WAL can't be removed (stuck replication slot, failing archive): `SELECT slot_name, active FROM pg_replication_slots;`. Fix the cause or drop an abandoned slot. Or **grow the volume**. | **Never** delete files in `pg_wal` by hand: it can corrupt the database |
| Cloud disk | **Grow the volume** (EBS modify-volume, then `growpart` + `xfs_growfs` / `resize2fs`) — usually online, no downtime | — |
| Inodes | Delete the millions of junk files (e.g. old sessions) in batches: `find /var/lib/php/sessions -type f -mmin +1440 -delete` | `rm *` in a dir with millions of files ("Argument list too long") |

### Permanent

- Log rotation everywhere (logrotate with `copytruncate` or a reopen signal; Docker `log-opts max-size`/`max-file`; journald `SystemMaxUse`).
- Separate disks/volumes for data, logs and the OS, so a log flood can't stop the database.
- Fix the cause: the debug log level, the bug creating files, the stuck replication slot.
- Kubernetes: set `ephemeral-storage` requests/limits on pods; let kubelet garbage-collect images.

---

## How to verify the fix

- `df -h` and `df -i` show healthy free space (e.g. < 80%), and the **growth rate** is back to normal (graph flat, not climbing).
- `lsof +L1` shows no big deleted files.
- The application's writes succeed (no new `ENOSPC` in logs); database accepts writes.
- Kubernetes: `DiskPressure=False`; evicted pods rescheduled; delete the old `Evicted` pod objects.
- The "disk will fill in N hours" alert has cleared.

---

## Prevention

- **Alert on prediction, not only percentage:** Prometheus `predict_linear(node_filesystem_avail_bytes[6h], 4*3600) < 0` → "will fill within 4 hours". Plus a static alert at 85–90%.
- **Alert on inodes** too (`node_filesystem_files_free`).
- Log rotation and retention as code (Ansible role, Docker `daemon.json`, Helm values).
- Dedicated volumes for databases and container storage.
- Monitor database-specific growth: WAL size, replication slot lag.
- Capacity review: growth trend per month.

---

## Follow-up questions

1. **Q: What was the real root cause, not just the trigger?**
   A: The trigger was "disk full". The root cause is whatever wrote the data and why it wasn't bounded: e.g. debug logging enabled in production with no rotation, or a replication slot for a decommissioned replica that kept WAL forever. Contributing factors are usually: no prediction alert, logs and data on the same disk.

2. **Q: How did you detect it, and why not earlier?**
   A: Typically by application errors (ENOSPC) or a 95% alert. It wasn't caught earlier because static percentage alerts fire too late when growth is fast; a `predict_linear` alert would have given hours of warning.

3. **Q: What did you check first, and why?**
   A: `df -h` and `df -i` to identify **which** filesystem and **which kind** of full, because the fix is completely different for blocks, inodes and deleted-open files, and a database disk is more urgent than `/tmp`.

4. **Q: How did you stop the impact quickly?**
   A: Freed space safely: truncate the runaway log (not `rm`), vacuum journald, prune unused images, or grow the cloud volume online. For a database, growing the volume is often the fastest and safest step.

5. **Q: `df` says 100% but `du` only finds 60%. Why?**
   A: A deleted file is still open by a process, so its blocks are still allocated. `lsof +L1` finds it. Truncate it via `/proc/<pid>/fd/<n>` or restart the process. Other causes: files hidden under a mount point (written before something was mounted on top), or reserved blocks.

6. **Q: How did you confirm it was fixed?**
   A: Free space and inodes healthy and stable, growth rate normal, no new ENOSPC errors, application/database writes succeeding, alerts cleared.

7. **Q: What if the fix didn't work — space filled up again in minutes?**
   A: Then something is actively writing fast. Find it with `iotop -o` or by watching `du` on the suspect directory, then stop or throttle that process (lower log level, stop the job). Meanwhile, grow the volume to buy time.

8. **Q: How do you prevent it?**
   A: Prediction-based and inode alerts, log rotation and retention everywhere as code, separate volumes for data/logs/OS, ephemeral-storage limits in Kubernetes, and monitoring database-specific growth (WAL, replication slots).

9. **Q: What if this happened on the primary database at 10× traffic?**
   A: Growth is faster, so time is shorter. Priority: keep the database accepting writes. Grow the volume immediately (online), and in parallel find the cause. Don't delete database files by hand. If the database already stopped, free space first, then start it and check its logs for recovery.

10. **Q: What if you can't SSH to the box?**
    A: Use the cloud console to grow the volume, use SSM Session Manager or a serial console, or in Kubernetes use `kubectl debug node/<node>`. If all else fails, detach the volume and attach it to a rescue instance.
