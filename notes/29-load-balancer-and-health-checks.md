# 29 · Load Balancer and Health-Check Failures

> **Level:** ★★ intermediate
>
> **You will learn:** how load balancers decide where to send traffic (**health checks**, **target groups**, **readiness**), the classic failures: **all targets unhealthy** (503), **health check too deep** (one dependency down → whole fleet "unhealthy"), **too shallow** (broken app still "healthy"), **wrong path/port/host header**, **deregistration and draining** mistakes causing errors on every deploy, **idle-timeout mismatches** causing 502s, and how to read **502 vs 503 vs 504**.
>
> **You need:** topics 01–28. Helpful: [SD 09 · Load balancing](../../System-Design/notes/09-load-balancing.md), [K8s 22 · Health probes](../../CKA-Certification-Course-2025/notes/22-health-probes.md), [K8s 70 · Ingress on EKS](../../CKA-Certification-Course-2025/notes/70-ingress-on-eks-path-routing.md), [Docker 32 · Restart and healthchecks](../../Docker/notes/32-restart-and-healthchecks.md).
>
> **Part 2 · Recurring failures · D. Network, DNS, certificates** · Previous: [28 · Network and firewall changes](28-network-and-firewall-changes.md) · Next: [30 · Connection-pool exhaustion](30-connection-pool-exhaustion.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A load balancer (LB) sends traffic only to targets that **pass health checks**. If the health check is wrong, the LB either sends traffic to broken servers or **removes all servers**, even healthy ones.

```
         health check: GET /healthz every 10s
LB ──────────────────────────────────────────► target 1  ✅ 200
   ──────────────────────────────────────────► target 2  ✅ 200
   ──────────────────────────────────────────► target 3  ❌ timeout → removed after N failures
all three ❌  →  LB returns 503 to every user, even if the app is mostly fine
```

| Code from the LB | Usually means |
|---|---|
| **502 Bad Gateway** | The LB reached a target but got an invalid/closed response (app crashed, connection reset, keep-alive timeout mismatch) |
| **503 Service Unavailable** | No healthy targets (or LB overloaded / target refused) |
| **504 Gateway Timeout** | Target didn't answer within the LB's timeout |

**Picture it.** A restaurant host seats guests only at tables marked "ready". If the waiter marks every table "not ready" because the dessert fridge is broken, nobody gets seated, even though the kitchen works fine.

**The one thing to know.** A health check for the **load balancer** should answer "**can this instance serve requests?**", not "**are all my dependencies healthy?**". Deep checks turn one dependency's failure into a total outage.

**Try it now:** find the health check path of one service and `curl -v` it. What does it actually check?

**You've got it if you can:** diagnose "all targets unhealthy" and explain 502 vs 503 vs 504.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Users | 502/503/504 from the LB/ingress (not from the app); often a plain LB error page |
| Cloud console | Target group: targets `unhealthy` with reason (`Health checks failed with these codes: [404]`, `Request timed out`) |
| Metrics | AWS ALB: `HTTPCode_ELB_5XX_Count` up while `HTTPCode_Target_5XX_Count` is flat (the LB is generating the errors); `HealthyHostCount` drops to 0 |
| Kubernetes | Endpoints/EndpointSlices empty (no Ready pods); ingress-controller logs `no endpoints available` / `upstream connect error` |
| Deploys | Short bursts of 502s during every rolling deploy |
| Flapping | Targets switching healthy/unhealthy constantly under load |

---

## Analogy

The **restaurant host** (Core).

---

## How it happens

| Cause | What happens |
|---|---|
| **Health check too deep** | `/healthz` checks DB, cache and three APIs; one API slows down → every instance fails the check → LB removes **all** targets → full outage caused by a partial problem |
| **Health check too shallow** | `/healthz` returns 200 even when the app can't serve (thread pool stuck, DB pool exhausted) → LB keeps sending traffic to broken instances |
| **Wrong path / port / host / code** | Check hits `/health` but the app moved it to `/healthz` (404); app requires a `Host` header; check expects 200 but app returns 301 redirect to HTTPS |
| **Security group** blocks the LB's health-check source | All targets unhealthy after an SG change |
| **Timeouts too aggressive under load** | Health checks time out when the app is busy → removed → remaining instances get more load → they fail too (cascade, topic 38) |
| **No draining / wrong order on deploy** | Pod killed before the LB stops sending traffic → in-flight requests fail (502) |
| **Keep-alive timeout mismatch** | App's idle keep-alive timeout **shorter** than the LB's → app closes a connection the LB is about to reuse → occasional 502 |
| **Readiness vs liveness confusion** (K8s) | Liveness probe checks dependencies → kubelet restarts all pods when a dependency is slow |

### Draining on deploy, done right (Kubernetes + LB)

```
1. Pod gets SIGTERM (deleted)
2. In parallel: pod removed from EndpointSlices → LB/ingress deregisters it (takes some seconds)
3. Problem: if the app exits immediately, requests still routed to it fail.
Fix: a preStop sleep (e.g. 5–15 s) so deregistration finishes first, then the app finishes in-flight requests,
     and terminationGracePeriodSeconds long enough for both.
```

---

## How to find it

**First:** who is returning the error — LB or app?

```bash
curl -sv https://shop.example.com/api/cart -o /dev/null 2>&1 | grep -E "< HTTP|< server"
```
```
< HTTP/2 503
< server: awselb/2.0
```
- `server: awselb/2.0` → the **ALB** generated the 503 (no healthy targets), not the app.

**Second:** target health and the reason.

```bash
aws elbv2 describe-target-health --target-group-arn <tg-arn> \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Reason,TargetHealth.Description]' --output table
```
```
| i-0a1 | unhealthy | Target.ResponseCodeMismatch | Health checks failed with these codes: [503] |
| i-0b2 | unhealthy | Target.ResponseCodeMismatch | Health checks failed with these codes: [503] |
```
- Instances answer, but `/healthz` returns **503** → the app itself says it's unhealthy. Why?

```bash
curl -s http://10.0.1.11:8080/healthz
```
```
{"status":"fail","checks":{"db":"ok","redis":"ok","recommendations-api":"timeout"}}
```
- The health check fails because an **optional** dependency (recommendations) times out → **too deep**.

**Kubernetes:**
```bash
kubectl get endpointslices -n shop -l kubernetes.io/service-name=web
kubectl describe pod <pod> | grep -A3 -E "Readiness|Liveness"
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=50 | grep -iE "upstream|no live"
```

**Third:** if 502s happen on deploys only, check draining: preStop hook, `terminationGracePeriodSeconds`, LB deregistration delay, and app keep-alive timeout vs LB idle timeout (ALB default idle timeout is 60 s).

---

## How to fix it

### Immediate

- **Deep check failing everywhere:** change the LB health check to a **shallow** path (e.g. `/livez` that only checks the process), or temporarily make the failing dependency non-fatal in the check (config flag). Restoring traffic to all instances ends the outage; the slow dependency then only degrades one feature.
- **Wrong path/code:** fix the health check settings (path, port, matcher codes).
- **SG change:** allow the LB's security group to the target port.
- **Flapping under load:** raise health check timeout/thresholds temporarily, add capacity.

### Permanent

- Separate endpoints:
  - `/livez` (liveness): is the process alive and not deadlocked? No dependency checks.
  - `/readyz` (readiness / LB): can **this** instance serve its **core** function? Include only **hard** dependencies, with short timeouts, and consider caching results.
  - Dependency health goes to **metrics/alerts**, not to the LB decision.
- Graceful shutdown: preStop delay, handle SIGTERM, drain connections, `terminationGracePeriodSeconds` > preStop + longest request.
- App keep-alive idle timeout **longer** than the LB's idle timeout.
- Some LBs **fail open** when all targets are unhealthy (e.g. AWS ALB routes to all targets if **all** are unhealthy; Envoy has "panic threshold"). Know your LB's behaviour.

---

## How to verify the fix

- `HealthyHostCount` = expected; target health all `healthy`.
- LB-generated 5xx at zero; user-facing success rate back in SLO.
- A test rolling deploy produces **zero** 502s.
- Simulate the optional dependency failing: instances stay healthy, only the feature degrades.

---

## Prevention

- Alerts: healthy host count below N or below 50%; LB-generated 5xx vs target 5xx.
- Health-check design reviewed in production readiness.
- Chaos test: make one dependency slow and check the LB doesn't remove everything.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: the LB health check included a call to an optional recommendations API. When that API slowed down, every instance failed the check and the LB removed all of them. Root cause: health check design mixing "can I serve?" with "are my dependencies OK?".

2. **Q: How did you detect it? Why not earlier?**
   A: SLO burn alert; healthy host count dropped to zero. Not caught earlier because the health check design was never tested against a dependency failure.

3. **Q: What did you check first?**
   A: Whether the LB or the app returned the error (server header and LB vs target 5xx metrics), then target health reasons.

4. **Q: How did you stop impact?**
   A: Pointed the target group health check at the shallow `/livez`, restoring all targets; the recommendations widget degraded gracefully.

5. **Q: Permanent fix?**
   A: Split liveness/readiness/dependency checks; dependencies became metrics with their own alerts.

6. **Q: How did you confirm?**
   A: Healthy targets restored; a chaos test that blocked the recommendations API left the LB healthy.

7. **Q: What if the shallow check made the LB send traffic to instances that really were broken?**
   A: That's the other risk. Readiness must still catch "this instance can't serve" (e.g. its DB pool is exhausted). Balance: include hard dependencies needed for the core path, with short timeouts; plus outlier detection (remove instances with high error rates) at the LB or mesh.

8. **Q: Why do we get a few 502s on every deploy?**
   A: Pods stop before the LB deregisters them, or keep-alive connections are closed by the app first. Fix with a preStop delay, graceful SIGTERM handling and a keep-alive timeout longer than the LB's.

9. **Q: What would you do differently?**
   A: Define a health-check standard for all services and test it with fault injection.

10. **Q: What if this happened at 10× traffic?**
    A: Health checks are more likely to time out because instances are busy, and removing instances pushes more load onto the rest — a cascade. Use generous health timeouts, autoscaling with headroom, load shedding in the app so it answers health checks even under load, and the LB's fail-open behaviour.
