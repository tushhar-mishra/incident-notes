# 36 · Retry Storms

> **Level:** ★★★ advanced
>
> **You will learn:** how **retries**, meant to make systems more reliable, can **multiply load** exactly when a system is weakest; the **amplification math** across layers; why the system often **can't recover on its own** even after the original cause is gone (a **metastable failure**); the fixes (**exponential backoff + jitter**, **retry budgets**, retrying at **one layer**, **circuit breakers**, **load shedding**), and how to spot a retry storm in metrics.
>
> **You need:** topics 01–35. Helpful: [SD 24 · Resilience patterns](../../System-Design/notes/24-resilience-patterns.md), [SD 25 · Rate limiting and backpressure](../../System-Design/notes/25-rate-limiting-and-backpressure.md).
>
> **Part 2 · Recurring failures · F. Overload and cascades** · Previous: [35 · Quotas and rate limits](35-quotas-and-rate-limits.md) · Next: [37 · Cache stampede and thundering herd](37-cache-stampede-thundering-herd.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** When a request fails, clients **retry**. If a service is slow because it's overloaded, retries **add more load**, making it slower, causing more timeouts, causing more retries.

```
Service slightly overloaded → some requests time out → clients retry
          ▲                                                 │
          └──────── load goes UP, more timeouts ◄───────────┘    (positive feedback loop)

Amplification across layers (3 retries = 4 attempts per layer):
  user → API (4×) → service (4×) → database (4×)  =  up to 4 × 4 × 4 = 64× load on the database
```

**Picture it.** A busy phone line. When people get a busy tone, they immediately redial, again and again. The more people redial, the busier the line, and nobody gets through — even after the original rush has passed.

**The one thing to know.** **Retry at one layer only, with exponential backoff + jitter and a retry budget.** Fast, unlimited, synchronized retries turn a small slowdown into a full outage.

**Try it now:** find the retry settings in one HTTP client you use (count, backoff, jitter, timeout). Then do the multiplication through every layer.

**You've got it if you can:** compute retry amplification, recognise a storm in metrics, and stop one.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: [68 · AWS us-east-1, December 2021](68-aws-us-east-1-2021.md), [53 · Google Cloud June 2025](53-google-cloud-service-control-2025.md) (missing randomized backoff on restart).

---

## Symptoms

| Where | What you see |
|---|---|
| Request rate | Incoming request rate to the struggling service **rises** while user traffic is flat (retries) |
| Ratio | Requests to downstream ÷ requests from users goes up (e.g. normally 1.0, now 3.5) |
| Errors | Timeouts and 503s rise together; retry counters (`client_retries_total`) spike |
| Recovery | You fix the original cause (rollback, restart), but the service **stays** overloaded |
| Restarts | Many instances restart at once and immediately get hammered (no warm-up) |
| Logs | The same request ID appears many times |

---

## Analogy

The **phone line redial** (Core). Backoff with jitter = everyone agrees to wait a **random** and **increasing** time before calling again, so calls spread out.

---

## How it happens

### Amplification

With `r` retries per layer (so `r+1` attempts) and `L` layers that each retry:

```
worst-case load on the bottom = (r + 1) ^ L
r = 3, L = 3   →   4^3 = 64×
r = 2, L = 4   →   3^4 = 81×
```

### Synchronization

Without jitter, all clients retry at the **same** moments (e.g. exactly 1 s, 2 s, 4 s after the failure), creating waves of load. **Jitter** (randomness) spreads them out.

### Metastable failure

A system is **metastable** when it has a stable "good" state and a stable "bad" state, and the bad state **sustains itself**. Retry storms are the classic example: the original trigger (a brief slowdown, a deploy, a network blip) is gone, but the retry load keeps the system overloaded. Recovery needs **active** action: cutting load below capacity so it can drain.

### Other self-sustaining loops

- **Timeouts shorter than the service's latency under load** → work is done, then thrown away, then retried.
- **Restart storms:** all instances restart together after a fix, each doing expensive startup (cache warm, connections), plus the backlog → overload again (Google Cloud June 2025: Service Control tasks restarting without randomized exponential backoff overloaded their backing store, topic 53).
- **Queue redelivery:** failed messages return to the queue immediately and are retried forever.

---

## How to find it

**First:** compare **user** traffic with **internal** traffic.

```
sum(rate(http_requests_total{job="edge"}[1m]))                # what users send
sum(rate(http_requests_total{job="orders-svc"}[1m]))          # what orders-svc receives
sum(rate(client_retries_total{target="orders-svc"}[1m]))      # retries, if instrumented
```
- Users flat at 2,000 req/s; orders-svc receiving 7,000 req/s → ~3.5× amplification.

**Second:** where are retries configured? Check every layer: client libraries, service mesh (Istio/Linkerd retry policies), API gateway, load balancer, SDKs (AWS SDK has its own retries), message consumers.

```bash
kubectl get virtualservice -A -o yaml | grep -A4 retries
```
```yaml
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure
```
- Mesh retries 3× **plus** app retries 3× → 16× from two layers.

**Third:** logs for repeated request IDs and timeouts:

```bash
kubectl logs -l app=orders-svc --since=5m | grep -o 'request_id=[a-z0-9-]*' | sort | uniq -c | sort -rn | head
```
```
      4 request_id=7f3c...
      4 request_id=a91e...
```
- Each request arriving 4 times.

---

## How to fix it

### Immediate: get load below capacity

1. **Shed load at the edge:** rate-limit or reject a percentage of traffic early (fast 429/503 is cheaper than slow timeouts); turn off non-critical features.
2. **Turn retries down/off** at inner layers (mesh retry policy, config flag) — this is the fastest way to cut amplification.
3. **Open circuit breakers** to the struggling dependency (fail fast instead of piling on).
4. **Scale** the struggling service only if the bottleneck is really there (and won't move to the DB).
5. **Bring capacity back gradually:** if you restart, do it in waves; let caches warm; admit traffic in steps (e.g. 10% → 25% → 50%).

### Permanent

- **Retry policy standard** (shared library or mesh config, one place only):
  - Retry only **idempotent** operations (safe to repeat).
  - **Exponential backoff with jitter** (e.g. base 100 ms, ×2 each try, random "full jitter", cap 10 s).
  - **Max 2–3 attempts**, and a **retry budget** (e.g. retries ≤ 10% of requests; stop retrying when the budget is used).
  - **One layer** retries (usually the outermost that knows the user request, or the innermost client — choose once).
  - Honour `Retry-After`.
- **Timeouts** that fit: each layer's timeout smaller than its caller's (deadline propagation).
- **Circuit breakers** and **load shedding** (admission control) in services.
- Startup with **randomized delay** and backoff when many tasks restart.
- Dead-letter queues for poison messages.

---

## How to verify the fix

- Internal/edge request ratio back to normal (≈ 1.0 for 1:1 calls).
- Retry rate low; error rate and latency in SLO.
- Load stays below capacity when traffic is restored to 100%.
- A fault-injection test (slow down the dependency) no longer causes amplification.

---

## Prevention

- Metrics: retries per client per target; the amplification ratio; circuit-breaker states.
- Review retry config in every layer during production readiness.
- Chaos tests: inject latency in a dependency and watch load upstream and downstream.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a 2-minute slowdown in the database (vacuum) caused timeouts; the app retried 3× and the mesh retried 3× on top, so the DB received up to 16× load and never recovered on its own. Root cause: retries at two layers with no backoff, jitter or budget.

2. **Q: How did you detect it? Why not earlier?**
   A: SLO burn; then internal request rate far above user traffic. The amplification ratio wasn't monitored.

3. **Q: What did you check first?**
   A: User traffic vs internal traffic to the struggling service: if internal is much higher, it's retries, and scaling alone won't fix it.

4. **Q: How did you stop impact?**
   A: Disabled mesh retries, shed 30% of traffic at the edge, waited for the DB to drain, then restored traffic in steps.

5. **Q: Permanent fix?**
   A: A single retry policy: app-level only, idempotent calls, backoff with full jitter, 2 retries max, retry budget; mesh retries off; deadlines propagated.

6. **Q: How did you confirm?**
   A: Ratio back to 1.0; a chaos test adding 500 ms DB latency produced a small, stable degradation instead of a storm.

7. **Q: What if shedding load didn't help?**
   A: Shed more, until the system drains; the "good" state only returns when load is clearly below capacity. Check for other loops: restart storms, queue redelivery, autoscaling adding instances that all warm up at once.

8. **Q: Why is jitter important?**
   A: Without it, all clients retry at the same instant, creating synchronized waves that hit at full force. Random delays spread retries over time.

9. **Q: What would you do differently?**
   A: Standardize retries in one library/config from the start, and instrument retries as a first-class metric.

10. **Q: What if this happened at 10× traffic?**
    A: The amplification starts sooner and from a higher base, so the system tips into the bad state faster. Admission control (reject early when overloaded) and retry budgets matter more than raw capacity.
