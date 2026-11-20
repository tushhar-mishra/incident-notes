# 23 · Bad Deploys

> **Level:** ★ foundation
>
> **You will learn:** why deploys are the **#1 trigger** of incidents, how to **prove** a deploy is the cause (timing, old vs new pods, canary comparison), how to **roll back** fast in Kubernetes, Argo CD, Helm, Jenkins and GitHub Actions, why rollback sometimes **fails** (database migrations, deleted images, config drift), why "**it passed in staging**", and the deployment strategies that make bad deploys small (**canary**, **blue/green**, **feature flags**, **automated analysis**).
>
> **You need:** topics 01–22. Helpful: [K8s 25 · Deployments](../../CKA-Certification-Course-2025/notes/25-deployments.md), [K8s 65 · Helm](../../CKA-Certification-Course-2025/notes/65-helm.md), [K8s 68 · GitOps](../../CKA-Certification-Course-2025/notes/68-gitops.md), [SD 34 · Deployment strategies](../../System-Design/notes/34-deployment-strategies.md).
>
> **Part 2 · Recurring failures · C. Changes** · Previous: [22 · Control-plane overload](22-control-plane-overload.md) · Next: [24 · Bad config pushes](24-bad-config-pushes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **bad deploy** is a new version of code that breaks something. The fix is almost always the same: **roll back first, debug later**. The real lesson is to make deploys **small and gradual**, so a bad one hurts few users for a short time.

```
All at once:  v1 ████████████ → v2 ████████████   bad v2 hits 100% of users
Canary:       v1 ███████████░ → v2 ░ (5%)  → analysis → 25% → 50% → 100%
                                 ▲ bad v2 hits 5% for 10 min, auto-rollback
```

**Picture it.** A restaurant testing a new recipe. Serving it to every table on day one is risky. Serving it to one table first, and watching their faces, is cheap insurance.

**The one thing to know.** If errors started within minutes of a deploy, **roll back without waiting for proof**. Then compare logs and metrics of old vs new version calmly.

**Try it now:** in your lab, deploy a broken image tag and roll back with `kubectl rollout undo`. Time it.

**You've got it if you can:** roll back in Kubernetes, Helm and Argo CD, and explain when rollback won't work.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Graphs | **Step change** in errors/latency starting within minutes of a deploy marker |
| Pods | New ReplicaSet pods crashing, not Ready, or Ready but returning errors |
| Logs | New error types (`NullPointerException`, `KeyError`, `500`) only on new pods |
| Canary analysis | Canary error rate/latency worse than baseline |
| Users | Broken feature, errors, slow pages; sometimes subtle (wrong prices, missing data) |
| Downstream | A dependency suddenly gets more load (new code calls it more often / retries) |

---

## Analogy

The **new recipe** (Core). Rollback = putting the old recipe back on the menu immediately; the chef works out what went wrong after service.

---

## How it happens

### Why bad code reaches production

| Reason | Example |
|---|---|
| **Staging ≠ production** | Different data size, config, traffic patterns, feature flags, dependencies, scale |
| **Tests don't cover it** | Edge cases, concurrency, error paths, real third-party behaviour |
| **Big batch** | 40 changes in one release; any one can break it; hard to find which |
| **All at once** | No canary; 100% of users get the bad version instantly |
| **Incompatible change** | API or schema change that old clients/consumers can't handle |
| **Hidden coupling** | Code depends on a config key, env var, or another service version (topic 19) |
| **Performance regression** | Works but is 3× slower; fine at low traffic, breaks at peak |

### Why rollback sometimes fails

- **Database migration** already ran and old code can't read the new schema (or new code wrote data old code can't parse). → Use **expand/contract** migrations ([SD 35](../../System-Design/notes/35-zero-downtime-database-changes.md)).
- **Old image deleted** by registry retention, or tag overwritten.
- **Config/secret changed** together with code and not rolled back with it.
- **External state:** messages already sent to a queue in a new format; caches filled with new-format data.
- **GitOps reverts your rollback:** you rolled back with `kubectl`, but Argo CD syncs Git (still on v2) and redeploys it.

---

## How to find it

**First:** does the start time line up with a deploy?

```bash
kubectl rollout history deploy/checkout -n shop
kubectl get rs -n shop -l app=checkout -o wide
```
```
REVISION  CHANGE-CAUSE
11        image=checkout:v2.13
12        image=checkout:v2.14

NAME                  DESIRED  CURRENT  READY  AGE  IMAGES
checkout-5b8c7d9e4f   0        0        0      3d   registry/checkout:v2.13
checkout-7d9f8b6c5d   6        6        6      9m   registry/checkout:v2.14
```
- v2.14 went out **9 minutes ago**; errors started 8 minutes ago → strong suspect.

Also: CI/CD history (Jenkins build, GitHub Actions run, Argo CD sync history), deploy annotations on Grafana dashboards.

**Second:** compare old vs new (if both exist — canary or mid-rollout).

```
sum by (version) (rate(http_requests_total{app="checkout",code=~"5.."}[5m]))
  / sum by (version) (rate(http_requests_total{app="checkout"}[5m]))
```
- New version 8% errors, old version 0.1% → the deploy is the cause.

```bash
kubectl logs -n shop -l app=checkout,pod-template-hash=7d9f8b6c5d --since=10m | grep -i error | sort | uniq -c | sort -rn | head
```

**Third:** what's in the release? `git log v2.13..v2.14 --oneline`. Look for migrations, config changes, dependency upgrades, new calls to other services.

---

## How to fix it

### Immediate: roll back

```bash
# Kubernetes (imperative; then fix Git if you use GitOps!)
kubectl rollout undo deploy/checkout -n shop            # to previous revision
kubectl rollout undo deploy/checkout -n shop --to-revision=11
kubectl rollout status deploy/checkout -n shop

# Helm
helm history checkout -n shop
helm rollback checkout 11 -n shop

# Argo CD (GitOps): preferred is git revert of the commit that bumped the version
git revert <commit> && git push      # Argo CD syncs back to v2.13
# (argocd app rollback <app> <id> works only with auto-sync disabled)

# Argo Rollouts canary
kubectl argo rollouts abort checkout -n shop
```

- **Jenkins / GitHub Actions:** re-run the deploy job of the last good version (parameterised with the previous image digest) ([Jenkins 64](../../Jenkins/notes/64-improving-the-pipeline.md), [GHA 53](../../GitHub-Actions-Basics-To-Production/notes/53-capstone-11-prod-deployment.md)).
- **Freeze deploys** for this service until it's understood.
- If rollback is impossible (migration): fix forward with the smallest possible change, reviewed by a second engineer; or turn the feature off with a flag.

### Permanent

- Fix the bug, **plus** a test for this class of bug.
- **Progressive delivery:** canary with automated analysis on error rate and latency and **automatic rollback** (Argo Rollouts, Flagger, Spinnaker).
- **Feature flags:** deploy code dark, enable gradually, kill-switch (topic 24).
- **Backward-compatible** migrations (expand/contract).
- **Smaller, more frequent releases.**
- Keep N previous images; deploy by digest.

---

## How to verify the fix

- Error rate and latency back to the pre-deploy baseline (compare same time yesterday).
- `kubectl get rs` shows the old ReplicaSet scaled up, new one at 0.
- Git/Argo CD state matches what's running (no drift that will redeploy the bad version).
- Business metric recovered (orders/minute), not only technical metrics.

---

## Prevention

- Deploy markers on every dashboard.
- Canary + automated analysis + auto-rollback for critical services.
- Readiness probes that check the app can really serve; `maxUnavailable: 0`.
- Production-like staging (data volume, config parity checks).
- Error budget policy: freeze risky deploys when the budget is exhausted.
- Track **change failure rate** (topic 10).

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: v2.14 added retries with no backoff; under a slow dependency they tripled traffic. The trigger was the deploy; the root cause was missing resilience standards for clients, plus the release going to 100% at once with no canary analysis.

2. **Q: How did you detect it? Why not earlier?**
   A: Error-budget burn alert ~7 minutes after deploy. A canary comparing new vs old would have caught it in the first minutes at 5% traffic.

3. **Q: What did you check first?**
   A: Whether the start time matched a deploy, because that's the most common cause and the fastest to mitigate.

4. **Q: How did you stop impact quickly?**
   A: Rolled back via git revert (GitOps), so Argo CD couldn't re-apply the bad version.

5. **Q: Permanent fix?**
   A: Fixed the code with a shared retry library, added a regression test, and moved the service to canary with automated analysis.

6. **Q: How did you confirm?**
   A: Errors back to baseline, running version matches Git, and the next release passed canary analysis.

7. **Q: What if rollback failed or wasn't possible?**
   A: If old code can't read the new schema, I'd turn the feature off by flag if possible, or fix forward with a minimal, reviewed hotfix. Long-term, migrations must be backward compatible so rollback is always possible.

8. **Q: Why did it pass in staging?**
   A: Staging lacked production's traffic volume and the dependency's slowness; errors only appeared under load. Fixes: load tests, canaries in production, and parity checks.

9. **Q: What would you do differently?**
   A: Ship smaller changes, behind a flag, with a canary; never 100% at once for critical paths.

10. **Q: What if the bad deploy happened during peak traffic at 10× load?**
    A: Impact is bigger and faster, so automatic rollback matters even more. Many teams also avoid deploying critical services during known peaks, or require a canary step that runs longer at peak.
