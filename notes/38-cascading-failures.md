# 38 · Cascading Failures

> **Level:** ★★★ advanced
>
> **You will learn:** how a failure in **one** component spreads to **others** until the whole system is down, the main spreading mechanisms (**resource exhaustion** from slow dependencies, **load redistribution** when instances fail, **retry storms**, **health-check removal**, **shared dependencies**), **circular dependencies** that block recovery, the defences (**timeouts, bulkheads, circuit breakers, load shedding, graceful degradation, static stability**), and how to **recover** a system that's collapsed (**cold start** in stages).
>
> **You need:** topics 01–37 (especially 29, 30, 36, 37). Helpful: [SD 24 · Resilience patterns](../../System-Design/notes/24-resilience-patterns.md), [SD 23 · Availability and the nines](../../System-Design/notes/23-availability-and-the-nines.md).
>
> **Part 2 · Recurring failures · F. Overload and cascades** · Previous: [37 · Cache stampede](37-cache-stampede-thundering-herd.md) · Next: [39 · Third-party and region outages](39-third-party-and-region-outages.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** In a **cascading failure**, one problem causes another, which causes another. The final outage is much bigger than the first problem.

```
Dependency C gets slow
   → Service B's threads wait on C → B's thread pool fills → B gets slow
      → Service A's calls to B time out → A retries → more load on B
         → B's health checks time out → LB removes B instances → fewer B instances take more load
            → remaining B instances die → A fails completely → users see a full outage
```

**Picture it.** Dominoes. Or a power grid: one power line fails, its load shifts to neighbouring lines, they overload and trip, their load shifts further, and a whole region goes dark.

**The one thing to know.** Cascades spread through **waiting** (no timeouts), **redistributing load** (losing instances), and **amplifying load** (retries). Every defence pattern breaks one of these links. **Set timeouts everywhere, isolate resources, and fail fast.**

**Try it now:** draw your service's dependencies. For each arrow, write the timeout and what happens if that dependency is down. Any "wait forever" or "crash" is a cascade path.

**You've got it if you can:** trace a cascade step by step and say which pattern would have stopped it at each step.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: [68 · AWS 2021](68-aws-us-east-1-2021.md), [69 · Canva 2024](69-canva-2024.md), [70 · Roblox 2021](70-roblox-2021.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Timeline | Problems appear in **sequence** across services (C first, then B, then A), minutes apart |
| Metrics | Thread/connection pools full in services that are themselves healthy; latency rising upstream of the real problem |
| LBs | Healthy host count dropping across tiers (topic 29) |
| Autoscaling | Scaling up but new instances fail too (overloaded on arrival, or cold) |
| Recovery | Fixing the original cause doesn't restore service (metastable, topic 36) |
| Dashboards | "Everything is red"; hard to see the first domino |

---

## Analogy

**Dominoes / the power grid** (Core).

---

## How it happens

### Spreading mechanisms

| Mechanism | How it spreads | Breaker |
|---|---|---|
| **Waiting on a slow dependency** | Threads/connections held → pool exhausted → caller slow | **Timeouts**, **bulkheads** (separate pools per dependency), async calls |
| **Load redistribution** | One instance dies → others get its share → they overload → die | **Headroom** (N+1/N+2), **load shedding**, slow-start |
| **Retries** | Failures multiply load | Backoff + jitter + budgets (topic 36) |
| **Health-check removal** | Deep checks remove healthy instances (topic 29) | Shallow LB checks, fail-open |
| **Shared dependency** | Auth, config, DNS, a database used by everything | Caching, static fallback, isolation |
| **Crash on bad input** | Every instance crashes on the same request ("query of death") | Input validation, isolate the bad request, per-request limits |
| **Autoscaler feedback** | Scale-up adds cold instances that need warm-up from the overloaded dependency | Pre-warmed capacity, scale on the right metric |

### Circular dependencies (why recovery is slow)

```
Service discovery (e.g. Consul) ──needed by──► monitoring, deploy tools, app
App + monitoring + deploy tools ──needed to fix──► service discovery
```
When the thing that's broken is needed by the tools you use to fix it, recovery slows dramatically. **Roblox 2021** (topic 70): their monitoring and other systems depended on the same Consul cluster that was failing. **Facebook 2021** (topic 59): internal tools needed the network that was down.

### Static stability

A system is **statically stable** if it keeps working **without needing to make changes** when a dependency fails. Example: an app keeps serving with its last known config if the config service is down, instead of failing to start. AWS uses this term in its architecture guidance.

---

## How to find it

**First:** find the **first domino**. Line up graphs of all services and find **which went bad first**.

```
Order of "latency p99 above normal":
  14:01:10  payments-db
  14:01:40  payments-svc
  14:02:30  checkout-svc
  14:03:00  api-gateway
```
- The cascade started at payments-db; everything upstream is a symptom.

Distributed tracing helps: in a slow trace, find the **deepest** span that is slow.

**Second:** at each tier, identify the **mechanism**: pool exhaustion (waiting), instance loss (redistribution), request ratio > 1 (retries).

```bash
kubectl top pods -n payments; kubectl get pods -n payments    # restarts? OOM? not ready?
```

**Third:** check shared dependencies (DNS, auth, config, service discovery) and the **tools** you'll need (monitoring, CI/CD, VPN, SSO): are they also affected?

---

## How to fix it

### Immediate

1. **Stop the spread:** open circuit breakers / disable calls to the failing dependency; degrade features ("recommendations unavailable").
2. **Reduce load below capacity:** shed at the edge; turn off non-critical traffic, batch jobs, retries.
3. **Fix or isolate the first domino** (roll back, fail over, restart, scale).
4. **Recover in stages (cold start):** bring the bottom tier up first, warm caches, then admit traffic gradually, tier by tier. Don't let 100% of traffic hit a freshly started system.

### Permanent

- **Timeouts on every remote call**, shorter than the caller's timeout.
- **Bulkheads:** separate thread/connection pools per dependency; separate node pools for critical services.
- **Circuit breakers** (Resilience4j, Envoy outlier detection/circuit breaking, Istio DestinationRule).
- **Load shedding / admission control:** reject early when overloaded.
- **Graceful degradation plans** per feature.
- **Headroom** to survive the loss of one instance/AZ.
- **Static stability:** cache config/credentials/discovery results; keep working on last known good data.
- **Break circular dependencies:** monitoring and incident tools must not depend on the systems they monitor (topic 44).

---

## How to verify the fix

- All tiers healthy; request ratios normal; no pools saturated.
- Recovered under **full** traffic (not just low traffic).
- Game day: kill or slow one dependency → impact limited to its features.

---

## Prevention

- Dependency map with timeouts and fallback behaviour per edge.
- Chaos engineering on dependencies (latency, errors, blackholes) ([SD 44](../../System-Design/notes/44-chaos-engineering-and-game-days.md)).
- Alerts on saturation (pools, queues) before errors.
- "Cold start" runbook for the whole system.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a slow third-party fraud check held checkout's worker threads (no timeout, shared thread pool), so checkout stopped serving **all** requests, including those that didn't need fraud checks. The trigger was the vendor's slowness; the root cause was missing timeouts and no bulkhead.

2. **Q: How did you detect it? Why not earlier?**
   A: SLO burn on checkout. Thread pool saturation wasn't alerted on, and vendor latency wasn't monitored.

3. **Q: What did you check first?**
   A: Which service went bad first (timeline across services/traces), because fixing a downstream symptom wastes time.

4. **Q: How did you stop impact quickly?**
   A: Disabled the fraud check call by flag (fail open for low-risk orders), which freed threads immediately.

5. **Q: Permanent fix?**
   A: 800 ms timeout, a separate pool for the vendor, a circuit breaker with a defined fallback, and vendor latency alerts.

6. **Q: How did you confirm?**
   A: A fault-injection test adding 10 s latency to the vendor left checkout healthy, with only the fraud feature degraded.

7. **Q: What if fixing the first domino didn't restore service?**
   A: The system is stuck in a bad state (retries, cold caches, lost instances). Shed load, restart in stages from the bottom up, and readmit traffic gradually.

8. **Q: What's a circular dependency, and why does it matter during incidents?**
   A: When the tools needed to fix a system depend on that system (monitoring on the same cluster, deploys needing the broken service discovery). It turns a short outage into a long one. Keep incident tooling independent.

9. **Q: What would you do differently?**
   A: Make "what happens if this dependency is slow or down?" a required question for every new dependency.

10. **Q: What if the cascade happened at 10× traffic?**
    A: Less headroom means each lost instance overloads the rest faster. Load shedding becomes essential; capacity plans must include losing an AZ at peak.
