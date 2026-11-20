# 14 · "Too Many Open Files"

> **Level:** ★★ intermediate
>
> **You will learn:** what a **file descriptor** (FD) is, why sockets count as "files", the **three limits** (per-process soft/hard **ulimit**, systemd **`LimitNOFILE`**, system-wide **`fs.file-max`**), why the error usually appears **under load**, how to tell a **leak** from a **limit that's too low**, the exact commands to inspect a running process, and how to fix it on Linux, systemd, Docker and Kubernetes.
>
> **You need:** topics 01–13. Helpful: [Linux 092 · Processes and system calls](../../linux-mastery-rhel/notes/092-processes-and-system-calls.md).
>
> **Part 2 · Recurring failures · A. Linux host resources** · Previous: [13 · Disk full](13-disk-full.md) · Next: [15 · High CPU, load and throttling](15-high-cpu-load-and-throttling.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** On Linux, every open file **and every network connection** uses a **file descriptor** (a small number the kernel gives the process). Each process has a **limit**. Hit it, and new connections or files fail with `EMFILE: Too many open files`.

```
process: nginx  (limit 1024)
  fd 0,1,2  stdin/stdout/stderr
  fd 3      access.log
  fd 4..1023 client sockets  ← traffic spike or a leak fills these
  fd 1024   accept() → EMFILE "Too many open files"
```

**Picture it.** A coat check with 1,024 hooks. Every guest (connection) takes a hook. When all hooks are full, new guests are turned away, even though the building has plenty of room.

**The one thing to know.** First ask: **is it a leak or a real need?** If the FD count **keeps climbing** even when traffic is flat, it's a **leak** (raising the limit only delays the crash). If it tracks traffic, the **limit is too low**.

**Try it now:** `ls /proc/$$/fd | wc -l` (how many FDs your shell has) and `ulimit -n` (your limit).

**You've got it if you can:** find a process's current FD count and its real limit, and raise the limit properly for a systemd service.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| App logs | `java.net.SocketException: Too many open files`, `accept4() failed (24: Too many open files)` (nginx), `EMFILE`, `OSError: [Errno 24] Too many open files` |
| System-wide | `VFS: file-max limit reached` in `dmesg` (`ENFILE`, error 23) |
| Users | Intermittent connection refused/reset, 502/503 from the load balancer, especially at peak |
| Graphs | FD count per process climbing in a straight line (leak) or hitting a flat ceiling (limit) |
| Other | Can't open log files, can't open new DB connections, health checks fail → restarts |

---

## Analogy

The **coat check** (Core). A **leak** is guests who leave without collecting their coat: the hooks fill up slowly even though there are fewer guests than hooks.

---

## How it happens

### What counts as a "file"

Regular files, directories, **TCP/UDP sockets**, Unix sockets, pipes, `epoll`/`inotify` handles, `/dev` devices. A busy proxy with 10,000 client connections and 10,000 upstream connections needs **20,000+** FDs.

### The limits

| Limit | Scope | Where it's set | Check |
|---|---|---|---|
| **Soft limit** (`RLIMIT_NOFILE`) | Per process; the one enforced | `ulimit -n`, `/etc/security/limits.conf` (PAM login sessions), systemd `LimitNOFILE=` | `cat /proc/<pid>/limits` |
| **Hard limit** | Per process; ceiling the soft limit can be raised to | same | same |
| **`fs.nr_open`** | Max a single process can ever have | sysctl | `sysctl fs.nr_open` |
| **`fs.file-max`** | System-wide total | sysctl | `cat /proc/sys/fs/file-nr` |

⚠️ **Common trap:** you raise the limit in `/etc/security/limits.conf`, but your service is started by **systemd**, which **ignores** that file. systemd services use `LimitNOFILE=` in the unit file. Containers inherit limits from the container runtime.

Default soft limit is often **1024** for interactive sessions; systemd's default for services is commonly 1024 soft / 524288 hard on modern systems (check your distro).

### Two root causes

```
Leak:     FD count  ────────────────────────/  climbs regardless of traffic
                                       /
                               /                 
Low limit: FD count  ─────/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾  follows traffic, flat-tops at the limit
```

**Leaks**: code opens files/sockets/HTTP clients and never closes them (missing `close()`, missing `finally`/`with`, creating a new HTTP client per request, connections stuck in `CLOSE_WAIT` because the app never closes its side).

---

## How to find it

**First:** confirm and identify the process.

```bash
grep -i "too many open files" /var/log/app/*.log | tail -3
journalctl -u nginx --since "15 min ago" | grep -i "open files"
```

**Second:** compare current count vs real limit for that **running** process.

```bash
PID=$(pgrep -o java)
ls /proc/$PID/fd | wc -l
grep "open files" /proc/$PID/limits
```
```
1021
Max open files            1024                 524288               files
```
- 1021 of 1024 → at the limit.
- The **soft** limit (1024) is what's enforced. The hard limit (524288) is the ceiling it could be raised to.

**Third:** what are the FDs? Leak or load?

```bash
sudo ls -l /proc/$PID/fd | awk '{print $NF}' | sed 's/[0-9]*//g' | sort | uniq -c | sort -n | tail
```
```
     12 /var/log/app/app.log
     40 anon_inode:[eventpoll]
    968 socket:[]
```
- Almost all are sockets → connections.

```bash
sudo ss -tanp | grep "pid=$PID" | awk '{print $1}' | sort | uniq -c
```
```
    120 ESTAB
    850 CLOSE-WAIT
```
- **850 in `CLOSE-WAIT`** = the remote side closed the connection, but **our app never called `close()`**. That's a **leak** in our code (or a library), not a traffic problem.

Watch the trend:
```bash
watch -n 5 "ls /proc/$PID/fd | wc -l"
```
Climbing while traffic is flat → leak.

System-wide:
```bash
cat /proc/sys/fs/file-nr
```
```
5120    0       9223372036854775807
```
- allocated, unused, max. If the first number is near the third → system-wide limit (rare on modern kernels with huge defaults).

---

## How to fix it

### Immediate

- **Restart** the affected process (rolling, one instance at a time) → FDs reset. Buys time; **not** a fix for a leak.
- **Raise the limit on the running process** without restart (Linux with util-linux `prlimit`):
  ```bash
  sudo prlimit --pid $PID --nofile=65536:65536
  ```
- Shift traffic away from the worst instances.

### Permanent

**systemd service:**
```bash
sudo systemctl edit myapp
```
```ini
[Service]
LimitNOFILE=65536
```
```bash
sudo systemctl daemon-reload && sudo systemctl restart myapp
grep "open files" /proc/$(systemctl show -p MainPID --value myapp)/limits
```

**Docker:** `docker run --ulimit nofile=65536:65536 …`, or `default-ulimits` in `daemon.json` ([Docker 34](../../Docker/notes/34-daemon-configuration.md)).

**Kubernetes:** containers inherit the container runtime's limits (containerd's systemd unit `LimitNOFILE`). There's no per-pod `ulimit` field in the Pod spec; set it at the node/runtime level, or raise the soft limit in the container's entrypoint up to the hard limit.

**nginx:** also set `worker_rlimit_nofile` and `worker_connections`.

**Fix the leak:** close resources (`with`/`try-with-resources`/`defer`), reuse one HTTP client with a connection pool, set idle timeouts, fix code that leaves sockets in `CLOSE_WAIT`.

---

## How to verify the fix

- `/proc/<pid>/limits` shows the new value **for the running process** (not just in the config file).
- FD count stays **flat** under steady traffic over hours (leak fixed) and stays well below the limit at peak.
- `CLOSE-WAIT` count stays near zero.
- No new "Too many open files" errors; error rate back in SLO.

---

## Prevention

- Export FD metrics: `process_open_fds` and `process_max_fds` (exposed by most Prometheus client libraries). Alert when `process_open_fds / process_max_fds > 0.8`.
- Alert on `CLOSE_WAIT` growth for long-running services.
- Set limits in code: systemd units, Docker daemon defaults, Ansible roles ([Ansible 17](../../Ansible/notes/17-modules-packages-services-users.md)).
- Load tests that run for hours (soak tests) catch leaks before production ([SD 31](../../System-Design/notes/31-capacity-planning-and-performance-testing.md)).

---

## Follow-up questions

1. **Q: What was the real root cause, not the trigger?**
   A: The trigger was the peak traffic that pushed FD count to the limit. The root cause was either a leak (sockets left in CLOSE_WAIT because the app never closed them) or a default limit of 1024 that was never sized for the service. Contributing: no FD metric or alert; the limit was set in limits.conf, which systemd ignores.

2. **Q: How did you detect it? Why not earlier?**
   A: Errors and 5xx at peak. Not earlier because we weren't graphing open FDs vs max; a ratio alert would have warned days before for a leak.

3. **Q: What did you check first, and why?**
   A: `/proc/<pid>/limits` and the FD count of the actual running process, because config files can lie (systemd ignores limits.conf) — only the running process's limits matter.

4. **Q: How did you stop the impact?**
   A: Rolling restart of the instances to reset FDs, and `prlimit` to raise the limit live on the remaining ones, while preparing the permanent change.

5. **Q: How did you fix it permanently?**
   A: `LimitNOFILE` in a systemd drop-in deployed by config management, and fixed the leak in code (reused a single HTTP client and closed responses), verified by a soak test.

6. **Q: How did you confirm it?**
   A: FD count flat over 24 hours at normal traffic and well under the limit at peak; CLOSE_WAIT near zero; no errors.

7. **Q: What if raising the limit didn't help?**
   A: Then it's a leak that fills any limit, or the limit didn't apply (check `/proc/<pid>/limits` again), or it's a different resource (ephemeral ports, conntrack table, system-wide file-max). I'd look at what the FDs are and at `dmesg`.

8. **Q: How would you prevent it?**
   A: FD-usage alerts, limits set as code, soak tests, code review rules for closing resources, and dashboards for socket states.

9. **Q: What would you do differently next time?**
   A: Add `process_open_fds / process_max_fds` to the standard service dashboard on day one, and check limits in the production readiness review.

10. **Q: What if it happened at 10× traffic on a reverse proxy?**
    A: At that scale, it may be a real need, not a leak: each client connection plus each upstream connection needs an FD. Raise limits (and `worker_connections`), enable upstream keepalive to reuse connections, and check related limits: ephemeral ports (`ip_local_port_range`), `conntrack` table size and `somaxconn`.
