# 15 · High CPU, Load and Throttling

> **Level:** ★★ intermediate
>
> **You will learn:** the difference between **CPU usage** and **load average**, why load can be high with **idle** CPUs (I/O wait, D-state), **steal time** on cloud VMs, **CPU throttling** in containers (CFS quota) and why it causes latency at "only 40% CPU", **burstable instance credits**, the commands to find which process/thread is burning CPU, and safe fixes.
>
> **You need:** topics 01–14. Helpful: [Linux 034 · top and load average](../../linux-mastery-rhel/notes/034-top-and-load-average.md), [Linux 097 · Namespaces and cgroups](../../linux-mastery-rhel/notes/097-namespaces-and-cgroups.md), [Docker 31 · Resource limits](../../Docker/notes/31-resource-limits.md), [K8s 41 · Requests, limits, QoS](../../CKA-Certification-Course-2025/notes/41-requests-limits-qos-quotas.md).
>
> **Part 2 · Recurring failures · A. Linux host resources** · Previous: [14 · Too many open files](14-too-many-open-files.md) · Next: [16 · Memory leaks and the OOM killer](16-memory-leaks-and-oom-killer.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** "The server is slow" can mean four different CPU problems:

```
1. Busy CPU         %us/%sy high              → some code is burning CPU
2. Waiting, not busy  load high, %wa high, CPU idle → stuck on disk/NFS (D-state)
3. Stolen CPU       %st high                  → the cloud host is giving your VM less CPU
4. Throttled CPU    container hits its quota   → latency spikes even at "40% CPU"
```

**Picture it.** A supermarket. Busy CPU = cashiers working flat out. I/O wait = cashiers idle because the card machine is slow. Steal = the manager keeps borrowing your cashiers for another shop. Throttling = cashiers are only allowed to work 20 minutes of every hour, so queues build even when they look half-idle.

**The one thing to know.** In containers, **CPU limits can cause latency without high average CPU**. Check `nr_throttled` / `container_cpu_cfs_throttled_periods_total` before blaming the code.

**Try it now:** `uptime; nproc` — is the 1-minute load higher than the CPU count?

**You've got it if you can:** tell these four apart with `top`, `vmstat` and the cgroup stats.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Alerts | High CPU, high load, latency SLO burn, HPA (Horizontal Pod Autoscaler) at max replicas |
| Users | Slow pages, timeouts; errors only at peak |
| Graphs | CPU at 100% (case 1); load high but CPU low (case 2); `st` > 5–10% (case 3); p99 latency spikes with CPU ~40–60% and throttling counter rising (case 4) |
| Shell | Commands slow to respond; SSH sluggish |
| Kubernetes | Liveness probes time out → restarts → even more load on the remaining pods |

---

## Analogy

The **supermarket** (Core).

---

## How it happens

### Load average

**Load average** = average number of tasks that are **running or waiting to run** (R state) **plus** tasks in **uninterruptible sleep** (D state, usually waiting for disk or NFS), over 1, 5 and 15 minutes. On Linux, D-state counts too, so load can be high with CPUs idle.

Rule of thumb: compare to the number of CPUs. Load 8 on 8 CPUs = fully used. Load 30 on 8 CPUs = big queue.

### Container CPU throttling (CFS quota)

In cgroups, a CPU **limit** is enforced by the **CFS** (Completely Fair Scheduler) **bandwidth control**: a **quota** of CPU time per **period** (default 100 ms).

```
Limit: 1 CPU  → quota 100 ms CPU time per 100 ms period
App with 8 threads handles a burst:
  8 threads × 12.5 ms = 100 ms quota used in the first 12.5 ms of the period
  → throttled (paused) for the remaining 87.5 ms
  → requests in that window wait ~87 ms extra → p99 latency spike
Average CPU over a minute: maybe only 40%.
```

That's why multi-threaded apps (JVM, Go, Node with worker threads) with tight limits show latency at moderate average CPU.

### Steal time

On VMs, `%st` = time your virtual CPU wanted to run but the hypervisor ran something else. Causes: noisy neighbours on shared hosts, or **burstable instances** (AWS T-family) that have used up their **CPU credits** and are capped to a baseline.

### Common causes of busy CPU

- A bad deploy (inefficient code, regex backtracking — see topic 50 Cloudflare 2019).
- Traffic increase or a bot.
- Garbage collection storms (JVM heap nearly full → GC runs constantly).
- Retry storms (topic 36).
- A runaway cron job or backup.
- Crypto-mining malware (unknown process using 100% CPU).

---

## How to find it

**First:** which kind of CPU problem?

```bash
uptime; nproc
vmstat 1 5
```
```
 14:03:11 up 41 days,  load average: 31.20, 18.40, 9.10
8
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
29  0      0 412000  50000 2100000    0    0     2    10 12000 30000 88 10  2  0  0
```
- Load 31 on 8 CPUs, rising (31 > 18 > 9) → getting worse.
- `r 29` → 29 runnable tasks, `us 88` → **busy CPU** in user code (case 1).
- If instead `b` (blocked) was high and `wa` high → I/O (case 2). If `st` high → steal (case 3).

**Second:** which process and which thread?

```bash
top -o %CPU          # press 1 for per-CPU, H for threads
pidstat -u 1 5
pidstat -t -p <PID> 1 3     # per-thread
```
For a JVM, map the hot thread ID (convert to hex) to a thread dump (`jstack <pid>` / `jcmd <pid> Thread.print`). For any process, `perf top -p <PID>` shows hot functions.

**Third (containers):** is it throttled?

```bash
# cgroup v2, from inside the container or the node
cat /sys/fs/cgroup/cpu.max         # "100000 100000" = 1 CPU limit
cat /sys/fs/cgroup/cpu.stat
```
```
usage_usec 912345678
nr_periods 360000
nr_throttled 92000
throttled_usec 4100000000
```
- `nr_throttled / nr_periods` = 92000 / 360000 ≈ **26% of periods throttled** → significant.
- Prometheus: `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])`.

```bash
kubectl top pods -n shop
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].resources}'
```

**Steal and credits (AWS):** `%st` in `top`; CloudWatch `CPUCreditBalance` near 0 for T-instances.

---

## How to fix it

### Immediate

| Case | Action |
|---|---|
| Busy CPU after a deploy | Roll back |
| Busy CPU from traffic | Scale out (`kubectl scale` / HPA max), rate-limit or block abusive clients |
| Runaway job | Stop or `renice +19` it; reschedule it off-peak |
| I/O wait | Find the process doing I/O (`iotop -o`, `pidstat -d`), pause it; check the disk/NFS |
| Steal / credits | Move to a non-burstable instance type, or enable unlimited mode (costs money) |
| Throttling | Raise or **remove the CPU limit** (keep the request), redeploy |
| Unknown process at 100% | Treat as **security incident**: isolate, don't just kill (topic 42/75) |

### Permanent

- Fix the inefficient code path (profile with `perf`, async-profiler, pprof).
- Right-size requests/limits from real usage. Many teams set **CPU requests but no CPU limits** for latency-sensitive services, relying on requests for fair sharing. (This is a common practice with trade-offs: noisy neighbours are possible; decide per cluster.)
- Match runtime threads to the CPU quota (e.g. Go `GOMAXPROCS` via `automaxprocs`; modern JVMs are container-aware).
- Autoscaling on the right signal (topic 12 in System Design: [SD 12](../../System-Design/notes/12-autoscaling.md)).

---

## How to verify the fix

- Load average below CPU count and falling; `r` column small.
- Throttled-period ratio near zero.
- **p99 latency** back in SLO (the user-facing proof).
- No probe failures/restarts.

---

## Prevention

- Alert on **latency SLO**, not raw CPU; use CPU/throttling as dashboard signals.
- Dashboard: CPU usage vs request vs limit, throttling ratio, steal.
- Load tests with production limits.
- Avoid burstable instances for steady production loads, or monitor credits.
- Canary deploys to catch CPU regressions on 5% of traffic.

---

## Follow-up questions

1. **Q: What was the root cause, not just "high CPU"?**
   A: High CPU is a symptom. The root cause might be a regex with catastrophic backtracking in a new release, GC thrashing from a memory leak, or a CPU limit that throttled a multi-threaded app. Each needs a different fix.

2. **Q: How did you detect it, and why not earlier?**
   A: The latency SLO alert. Throttling wasn't visible earlier because average CPU looked fine; we weren't graphing the throttled-period ratio.

3. **Q: What did you check first?**
   A: `vmstat`/`top` to classify: user CPU, I/O wait, steal or throttling. That decides everything else.

4. **Q: How did you stop the impact?**
   A: Rolled back the release (if a deploy lined up), or scaled out and removed the CPU limit on the throttled deployment.

5. **Q: How did you fix it permanently?**
   A: Fixed the code path found with a profiler; set requests from measured usage; aligned runtime thread count with CPU allocation.

6. **Q: How did you confirm?**
   A: p99 back in SLO at the next peak; throttling ratio near zero; load below core count.

7. **Q: What if scaling out didn't help?**
   A: Then the bottleneck isn't these pods' CPU: maybe a downstream (DB) or a lock. Check where time is spent with traces; check DB saturation.

8. **Q: Load average is 40 on an 8-CPU machine, but the CPUs are 90% idle. Explain.**
   A: Tasks in D state (uninterruptible sleep), usually waiting on slow disk or a hung NFS mount, count toward load. Check `vmstat` `b` and `wa`, `ps -eo state,pid,cmd | grep ^D`, and the storage.

9. **Q: What if this happened during a deployment?**
   A: Rolling updates temporarily reduce capacity and start cold instances (JIT warm-up, empty caches), which use more CPU. Use `maxUnavailable: 0`, `maxSurge`, readiness probes, and warm-up before taking traffic.

10. **Q: What would you do differently?**
    A: Include throttling and steal in the default dashboards, and require load testing with the production resource settings.
