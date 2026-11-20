# 16 · Memory Leaks and the OOM Killer

> **Level:** ★★ intermediate
>
> **You will learn:** how Linux uses memory (**RSS**, **page cache**, **available**), what the **OOM (Out Of Memory) killer** is, how it **chooses a victim** (`oom_score`, `oom_score_adj`), the difference between a **host OOM** and a **cgroup (container) OOM**, how a **memory leak** looks on a graph, the commands to prove it, and how to fix it without losing the evidence.
>
> **You need:** topics 01–15. Helpful: [Linux 093 · Memory management](../../linux-mastery-rhel/notes/093-memory-management.md), [Linux 050 · Swap](../../linux-mastery-rhel/notes/050-swap.md).
>
> **Part 2 · Recurring failures · A. Linux host resources** · Previous: [15 · High CPU](15-high-cpu-load-and-throttling.md) · Next: [17 · Clock skew and NTP](17-clock-skew-and-ntp.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** When a Linux machine (or a cgroup/container) runs out of memory, the kernel's **OOM killer** picks a process and **kills it** with `SIGKILL` to save the system. A **memory leak** is the most common reason: memory use grows and never comes back down.

```
Memory used
   ▲                      /|        /|        /|      ← leak: sawtooth
   │                    /  |      /  |      /  |        (grow → OOM kill → restart → grow)
   │                  /    |    /    |    /    |
   │                /      |  /      |  /      |
   └──────────────────────────────────────────────► time
```

**Picture it.** A bathtub with the plug half in. Water (memory) keeps rising slowly. When it overflows, the house's safety system (OOM killer) pulls out the biggest bucket of water it can find, often the most important one.

**The one thing to know.** "**free**" memory being low is normal (Linux uses spare RAM as cache). Look at **"available"** and at **per-process RSS growing over time**.

**Try it now:** `free -m` and `dmesg -T | grep -i "killed process"`.

**You've got it if you can:** prove an OOM kill happened, name the victim, and show whether it was host-level or container-level.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Kernel log | `Out of memory: Killed process 4211 (java) total-vm:… anon-rss:…` or `Memory cgroup out of memory: Killed process …` |
| Service | Process disappears and restarts; systemd: `Main process exited, code=killed, status=9/KILL` |
| Containers | Exit code **137** (128 + 9 = SIGKILL); `OOMKilled: true` (topic 18) |
| Graphs | Sawtooth memory pattern; restarts at regular intervals; swap usage rising; latency rising before each crash (GC pressure, swapping) |
| Users | Intermittent errors every N hours; slow responses before each crash |
| Whole host | If a critical process (sshd, kubelet, database) was killed: host unreachable or node NotReady |

---

## Analogy

The **bathtub** (Core). Swap is like a bucket next to the bath: it buys time but everything gets very slow while you bail.

---

## How it happens

### How Linux counts memory

```bash
free -m
```
```
               total        used        free      shared  buff/cache   available
Mem:           15880       14100         210          45        1570        1380
Swap:           2047        1900         147
```
- `free 210` → looks scary, but Linux uses free RAM as page cache. Don't panic about this number.
- `available 1380` → what new programs can actually get (free + reclaimable cache). **This is the one to watch.**
- `Swap used 1900 of 2047` → the system is under real memory pressure.

**RSS** (Resident Set Size) = the physical memory a process is using right now. A leak = RSS growing without limit.

### Why memory runs out

- **Leak:** objects allocated and never freed (unbounded caches, growing maps/lists, listeners never removed, native memory in a library).
- **Real need:** traffic grew; a big request (export of 10 million rows into memory).
- **Misconfiguration:** JVM `-Xmx` larger than the container limit; too many worker processes for the RAM.
- **Neighbour:** another process on the host is the one leaking; your process got killed because it's the biggest.

### How the OOM killer chooses

Each process has an `oom_score` (roughly proportional to its memory use) adjusted by `oom_score_adj` (-1000 to +1000). The kernel kills the process with the **highest** score. `-1000` = never kill. Kubernetes sets `oom_score_adj` based on QoS class, so **BestEffort** pods die first and **Guaranteed** pods last (topic 18).

### Host OOM vs cgroup OOM

| | Host (global) OOM | cgroup (container) OOM |
|---|---|---|
| Trigger | The whole machine is out of memory | One cgroup hit its own `memory.max` limit |
| Log line | `Out of memory: Killed process …` | `Memory cgroup out of memory: Killed process …` |
| Victim | Highest score anywhere on the host | A process **inside** that cgroup |
| Meaning | The host is overcommitted | That container's limit is too small, or it leaks |

---

## How to find it

**First:** did an OOM kill happen, and who died?

```bash
sudo dmesg -T | grep -iE "killed process|out of memory" | tail
```
```
[Thu Nov 20 03:12:44 2026] Memory cgroup out of memory: Killed process 4211 (java) total-vm:8123456kB, anon-rss:2091520kB, file-rss:10240kB, shmem-rss:0kB, UID:1000 pgtables:5000kB oom_score_adj:937
```
- `Memory cgroup` → container limit, not the host.
- `java`, `anon-rss` ≈ 2 GB → the JVM used ~2 GB of anonymous (heap/native) memory.
- `oom_score_adj:937` → in Kubernetes, a **Burstable** pod with a small memory request (Burstable pods get 2–999, higher for smaller requests). **BestEffort** pods get 1000 (die first); **Guaranteed** pods get -997 (die last).

```bash
journalctl -k --since "1 hour ago" | grep -i oom
systemctl status myapp       # "code=killed, status=9/KILL"
```

**Second:** is it a leak? Look at the trend.

```bash
pidstat -r -p <PID> 60 10          # RSS every minute
ps -o pid,rss,etime,cmd -p <PID>
```
```
  PID   RSS     ELAPSED CMD
 4211 1802340   05:10:12 java -jar app.jar
```
Compare over time: RSS rising steadily with flat traffic = leak. Prometheus: `process_resident_memory_bytes` or `container_memory_working_set_bytes` — a straight upward line between restarts.

**Third:** what is leaking? Capture evidence **before** restarting:

| Runtime | Tool |
|---|---|
| JVM | `jcmd <pid> GC.heap_info`; heap dump `jcmd <pid> GC.heap_dump /tmp/heap.hprof` (⚠️ pauses the app, large file); `-XX:+HeapDumpOnOutOfMemoryError` |
| Go | `pprof` heap profile (`/debug/pprof/heap`) |
| Python | `tracemalloc`, `memray` |
| Node.js | `--heapsnapshot-signal`, `--inspect` + Chrome DevTools |
| Native | `pmap -x <pid>`, `valgrind`, `heaptrack` |

Compare two snapshots taken some time apart: the object type that grew is the leak.

---

## How to fix it

### Immediate

- **Rolling restart** of the leaking service (memory resets). Schedule it more often than the crash interval as a stopgap (e.g. a nightly restart) — **only** while the real fix is coming.
- **Roll back** if the leak started with a release.
- **Raise the memory limit** temporarily if it's a real need, or move the workload to a bigger node.
- If the **host** OOM-killed a critical process (kubelet, sshd, database): restore it, and move noisy processes away.

### Permanent

- Fix the leak (bound caches with max size/TTL; close resources; remove listeners).
- Size runtimes **inside** the limit: JVM `-XX:MaxRAMPercentage=75` (leaves room for non-heap memory), Node `--max-old-space-size`.
- Kubernetes: set memory **request = limit** for critical services (Guaranteed QoS); protect system daemons with `kube-reserved`/`system-reserved`.
- Decide on swap deliberately (disabled on many Kubernetes nodes by default; swap can hide leaks behind slowness).

---

## How to verify the fix

- Memory graph is **flat** (or periodic with a stable ceiling, e.g. GC sawtooth that returns to the same baseline) over at least one full traffic cycle (24h).
- No OOM lines in `dmesg`; restart count stable.
- Latency stable (no slow-down before crashes).

---

## Prevention

- Alerts: container memory working set > 90% of limit; restarts > N per hour; OOM events (`container_oom_events_total` / kube-state-metrics `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}`).
- Soak tests (hours-long load tests) to catch slow leaks.
- Heap dump on OOM enabled by default (to a volume with space!).
- Canary deploys watching memory growth rate, not just errors.

---

## Follow-up questions

1. **Q: What was the root cause, not the trigger?**
   A: The trigger was the OOM kill. The root cause was, for example, an in-memory cache with no size limit that grew with every unique customer ID, combined with a container limit and no alert on memory growth.

2. **Q: How did you detect it, and why not earlier?**
   A: Repeated restarts and intermittent errors. It wasn't caught earlier because the service "self-healed" by restarting, and we only alerted on availability, not on restarts or memory trend.

3. **Q: What did you check first?**
   A: `dmesg` for the OOM line, to confirm it was an OOM kill (not a crash), see the victim, and tell host OOM from cgroup OOM.

4. **Q: How did you stop the impact quickly?**
   A: Rolled back the release that introduced the cache and, while preparing that, did rolling restarts to keep instances away from the limit.

5. **Q: How did you fix it permanently?**
   A: Bounded the cache (LRU with max entries and TTL), aligned JVM heap with the container limit, and added memory-growth alerts.

6. **Q: How did you confirm?**
   A: 48 hours with flat memory and zero OOM events under normal and peak load.

7. **Q: What if restarting didn't help — it OOMs immediately after start?**
   A: Then it isn't a slow leak: the working set at startup is larger than the limit (bigger data set, a config change, a large request on startup). Raise the limit temporarily, check what changed (data size, config), and look at the heap at startup.

8. **Q: How do you avoid losing evidence?**
   A: Enable heap dump on OOM in advance; before a manual restart, take a heap/profile snapshot from one instance and leave the others to restart.

9. **Q: What if the OOM killer killed the database instead of the leaking app?**
   A: That's the host-level OOM picking the largest process. Immediate: restart the DB and stop the leaking app. Permanent: protect the DB (`oom_score_adj` negative, or `OOMScoreAdjust=` in systemd), put memory limits on the other processes, and don't co-locate them.

10. **Q: What would you do differently?**
    A: Alert on restarts and memory growth, not just availability, and run soak tests before releases with new caching code.
