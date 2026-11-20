# 08 · Debugging Under Pressure

> **Level:** ★★ intermediate
>
> **You will learn:** a **method** for finding the cause when the first rollback didn't help: the **four golden signals**, the **USE** method (for resources) and the **RED** method (for services), **"follow the request"**, **hypothesis → test → conclude**, **bisecting** the system, how to read **graphs** (start time, shape, correlation), and a **core command kit** for Linux, Docker, Kubernetes and the cloud, each with what its output tells you.
>
> **You need:** topics 01–07.
>
> **Part 1 · Incident basics** · Previous: [07 · The first 15 minutes](07-the-first-15-minutes.md) · Next: [09 · Runbooks and playbooks](09-runbooks-and-playbooks.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Debugging is not "look at random things until something looks weird". It's a loop:

```
Observe (symptom, start time) → Hypothesise (2–3 candidates) → Test (one cheap check each)
      ▲                                                                │
      └──────────── rule out / confirm, narrow the search ◄────────────┘
```

Use two checklists to make sure you don't skip anything:
- **RED** for every **service** (request-driven thing): **R**ate, **E**rrors, **D**uration.
- **USE** for every **resource** (CPU, memory, disk, network, connection pool): **U**tilisation, **S**aturation, **E**rrors.

**Picture it.** A doctor: symptoms → list of possible illnesses → one test each → narrow down. They don't order every test in the hospital.

**The one thing to know.** **Follow the request.** Draw the path (user → DNS → CDN → load balancer → app → cache → DB → dependency) and check each hop. The failure is at the first hop that looks wrong.

**Try it now:** for one service you know, draw the request path and write the one command that checks each hop.

**You've got it if you can:** given "latency doubled", list three hypotheses and the one check that would confirm or rule out each.

> 📖 **That's the core.** Everything below is the **full reference**. Tool-specific deep dives: [Linux 102 · Troubleshooting method](../../linux-mastery-rhel/notes/102-troubleshooting-method.md), [Docker 50 · Debugging method](../../Docker/notes/50-debugging-method.md), [K8s 81 · Troubleshooting method](../../CKA-Certification-Course-2025/notes/81-troubleshooting-method.md), [SD 58 · Troubleshooting at scale](../../System-Design/notes/58-troubleshooting-at-scale.md).

---

## What it is

### The four golden signals (Google SRE book)

| Signal | Meaning | Example metric |
|---|---|---|
| **Latency** | How long requests take (separate successful vs failed) | p50/p95/p99 request duration |
| **Traffic** | How much demand | Requests per second |
| **Errors** | Rate of failed requests | 5xx per second, % failed |
| **Saturation** | How "full" the service is | CPU %, queue depth, pool in use / pool size |

### RED and USE

| Method | For | R / U | E / S | D / E |
|---|---|---|---|---|
| **RED** (Tom Wilkie) | Services | **Rate**: requests/s | **Errors**: failed requests/s | **Duration**: latency distribution |
| **USE** (Brendan Gregg) | Resources | **Utilisation**: % of time busy | **Saturation**: extra work waiting (queue) | **Errors**: error events |

**Saturation** is the one people forget. A CPU at 70% utilisation may still have a long run queue (load average much higher than core count). A connection pool can be 100% used with 500 requests **waiting** for a connection.

---

## Why it exists

Under pressure, people look at whatever is in front of them. A method makes sure that:
- You check **all** the likely places, not just your favourite.
- You **narrow down** quickly, by ruling things out.
- Several people can split the work ("you check DNS and LB, I check DB").

---

## Analogy

**Finding a leak in a garden hose.** You don't inspect every centimetre. You squeeze the hose halfway: is the water still coming out past that point? If yes, the leak is after; if no, before. Each squeeze halves the search. That's **bisecting**.

---

## How it works

### 1. Read the graph properly

Three questions about any bad graph:

| Question | Why it matters |
|---|---|
| **When exactly did it start?** | Lines it up with changes (topic 07) |
| **What shape is it?** | **Step** (instant jump) → a change or a hard failure. **Ramp** (slowly rising) → leak, growth, backlog. **Sawtooth** → crash-restart loop or GC. **Periodic spikes** → cron job, batch, cache expiry. |
| **What else moved at the same time?** | Correlation points to the cause (latency up exactly when DB CPU went up) |

### 2. Hypothesise and test

Write them down in the channel:

```
H1: payments-svc deploy at 13:58         test: compare errors on old vs new pods
H2: payment provider degraded            test: provider latency metric / status page
H3: DB connection pool exhausted         test: pool in-use metric; app logs "timeout acquiring connection"
```

One cheap test per hypothesis. **Rule things out** as fast as you rule them in.

### 3. Follow the request (bisect)

```
User → DNS → CDN → Load balancer → Ingress → Service → Pod (app) → Cache → DB
                                                                  ↘ external API
```

Test each hop from **outside in** (or from the middle, to bisect):

```bash
dig +short shop.example.com                         # DNS: does the name resolve? to what?
curl -sv https://shop.example.com/healthz -o /dev/null   # TLS + LB + app, from outside
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' https://shop.example.com/api/cart
kubectl -n shop exec deploy/checkout -- curl -s localhost:8080/healthz   # skip LB/ingress
kubectl -n shop exec deploy/checkout -- nc -zv postgres 5432            # app → DB reachability
```

If the app answers from inside the pod but not through the load balancer, the problem is **between** them.

### 4. Compare good vs bad

The fastest clue is often a **difference**:
- Old pods vs new pods (after a canary).
- Region A (fine) vs region B (broken).
- Customer X (broken) vs everyone else.
- Yesterday's graph vs today's at the same time.

### The core command kit

All sample outputs are illustrative.

**Linux host (60-second check)** — see [Linux 103 · Performance and observability](../../linux-mastery-rhel/notes/103-performance-and-observability.md):

```bash
uptime                 # load averages vs CPU count
dmesg -T | tail        # kernel errors: OOM kills, disk errors, network drops
vmstat 1 5             # r = run queue, si/so = swapping, wa = waiting on I/O
mpstat -P ALL 1 3      # per-CPU: is one core pinned at 100%?
pidstat 1 3            # which process uses CPU
iostat -xz 1 3         # disk: %util, await (latency)
free -m                # memory; "available" is the real free number
sar -n DEV 1 3         # network throughput
ss -s                  # socket summary: many TIME_WAIT? many established?
df -h; df -i           # disk space and inodes
```

Sample `vmstat 1 5`:
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
14  0      0 152340  20480 812300    0    0     5    40 9800 15000 92  7  1  0  0
```
- `r 14` on a 4-core machine → 14 runnable threads waiting for 4 CPUs = **CPU saturation**.
- `us 92` → user code is burning CPU (an application issue, not the kernel).
- `wa 0` → not waiting on disk. `si/so 0` → not swapping.
- `st 0` → no CPU stolen by the hypervisor (topic 15).

**Docker** — see [Docker 51 · Debugging tools](../../Docker/notes/51-debugging-tools.md):

```bash
docker ps -a --format 'table {{.Names}}\t{{.Status}}'       # restarts? "Exited (137)"?
docker logs --since 15m --tail 100 <name>
docker inspect <name> -f '{{.State.OOMKilled}} {{.RestartCount}} {{.State.ExitCode}}'
docker stats --no-stream
```

**Kubernetes** — see [K8s 83 · Data plane troubleshooting](../../CKA-Certification-Course-2025/notes/83-data-plane-troubleshooting.md):

```bash
kubectl get pods -n shop -o wide            # STATUS, RESTARTS, which node
kubectl describe pod <pod> -n shop          # Events at the bottom: the "why"
kubectl logs <pod> -n shop --previous       # logs from the crashed container
kubectl get events -n shop --sort-by=.lastTimestamp | tail -20
kubectl top pods -n shop; kubectl top nodes
kubectl get nodes                           # any NotReady?
kubectl get endpointslices -n shop          # does the Service have ready endpoints?
```

Sample `kubectl get pods -n shop`:
```
NAME                        READY   STATUS             RESTARTS        AGE
checkout-7d9f8b6c5d-2xkqp   0/1     CrashLoopBackOff   6 (2m ago)      9m
checkout-7d9f8b6c5d-8hrtz   0/1     CrashLoopBackOff   6 (90s ago)     9m
checkout-5b8c7d9e4f-lmn2p   1/1     Running            0               3d
```
- Two pods with a **new** ReplicaSet hash (`7d9f8b6c5d`) are crashing; the old one (`5b8c7d9e4f`) is fine → the **new version** is bad. Roll back (topic 19).

**Cloud (AWS examples)**:

```bash
aws cloudtrail lookup-events --start-time "$(date -u -d '-1 hour' +%FT%TZ)" \
  --max-results 50 --query 'Events[].[EventTime,EventName,Username]' --output table   # who changed what
aws health describe-events --region us-east-1     # AWS-side issues on your account (needs Business/Enterprise support for the API)
aws cloudwatch get-metric-statistics ...          # LB 5xx, target response time
```

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Looking only where the light is** | You check your own service for an hour; DNS was broken | Follow the request hop by hop |
| **Confirmation bias** | Only looking for evidence that your theory is right | Try to **disprove** your theory first |
| **Averages hide it** | Average latency fine; p99 terrible for one region | Look at percentiles and break down by dimension |
| **Correlation ≠ cause** | DB CPU up and errors up, but DB CPU went up **because** of retries | Check which moved **first** |
| **Missing saturation** | Utilisation 60%, but queues are long | USE: always check saturation |
| **No baseline** | "Is 2,000 connections normal?" | Compare with yesterday / last week |

---

## Where you'll use it

- Every Part 2 scenario's "How to find it" section uses this kit.
- **System Design:** [SD 27 · Observability](../../System-Design/notes/27-observability.md) explains metrics, logs, traces and why percentiles matter.

---

## Quick recap

- Loop: observe → hypothesise (2–3) → test (one cheap check each) → narrow.
- Golden signals: latency, traffic, errors, saturation. RED for services, USE for resources.
- Read the graph: start time, shape (step, ramp, sawtooth, periodic), what moved together.
- Follow the request hop by hop; compare good vs bad.
- Have a 60-second command kit for each layer.

### Interview questions

1. **Q: Explain the USE and RED methods.**
   A: USE (Utilisation, Saturation, Errors) is for resources like CPU, memory, disks and connection pools: how busy, how much work is queued, and error events. RED (Rate, Errors, Duration) is for request-driven services: requests per second, failures, and latency distribution. RED tells you **that** a service is unhealthy; USE helps find **which resource** is the bottleneck.

2. **Q: Latency doubled 10 minutes ago, nothing was deployed. How do you approach it?**
   A: Confirm the scope (all endpoints or one, all regions or one, p50 or only p99). List hypotheses: a downstream dependency got slower, a resource is saturated (DB, pool, CPU throttling), traffic changed. Test each cheaply: per-dependency latency in traces or metrics, saturation metrics (pool waiting, run queue, throttling), request rate and mix. Compare with the same time yesterday. Follow the request to find the first slow hop.

3. **Q: DB CPU and API errors went up at the same time. Is the database the cause?**
   A: Not necessarily. Correlation isn't causation. Check which one moved first, and why the DB is busy: if the API started retrying on timeouts, the extra DB load may be a **result**. Look at query rate per caller, slow query logs, and whether the request rate to the DB rose before or after the errors.

4. **Q: What does saturation mean, and why is it important?**
   A: It's work that is waiting because a resource is full: run queue length for CPU, I/O queue for disks, waiters for a connection pool. Utilisation can look moderate while saturation is high, and saturation is what turns into latency for users. It's also often the earliest warning sign.
