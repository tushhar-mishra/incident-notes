# 77 · Drill: 5xx Spike Right After a Deploy

> **Level:** ★ foundation
>
> **You will practise:** the most common incident of all — errors jump minutes after a deploy. Correlating with the change, comparing old vs new pods, rolling back in **Kubernetes**, **Helm** and **GitOps**, and making sure the rollback **sticks**.
>
> **You need:** topics 07, 19, 23, 76.
>
> **Part 4 · On-call drills** · Previous: [76 · How to run drills](76-how-to-run-drills.md) · Next: [78 · Disk at 95% on the DB host](78-drill-disk-95-database.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **03:07 — PAGE:** `CheckoutErrorBudgetFastBurn — checkout 5xx at 7.8% (SLO allows 0.1%), started 03:01`

**The idea.** Something changed; errors followed. **Roll back first**, then compare old vs new calmly.

```
03:01 errors start  ←→  deploy history: checkout v2.14 rolled out 02:58
→ roll back now → verify error rate → THEN compare v2.13 vs v2.14 → fix forward properly later
```

**The one thing to know.** In GitOps, `kubectl rollout undo` is **temporary** — Argo CD will re-apply Git. Roll back **in Git**.

> 📖 Read only **Symptoms**, then stop and write your first three checks before continuing.

---

## Symptoms

- Burn-rate page; dashboard shows 5xx jumping from ~0 to 7–8% at 03:01 (a **step**).
- Load balancer / ingress logs: 500s from `checkout` only.
- Users: "payment failed, please try again".
- A deploy notification in #deploys at 02:58 (if you look).

## Analogy

A restaurant changes the recipe at 7 PM; at 7:05 diners start sending plates back. You put the old recipe back first and investigate in the kitchen afterwards.

## How it happens

A new version has a bug that only shows with production traffic/data/config: a missing config key, a changed API contract, a slow query, a null pointer on real data (topic 23). Rolling updates replace pods gradually, so errors climb as more new pods take traffic.

### Lab setup (break it)

```bash
kubectl -n shop create deployment checkout --image=nginx:1.27 --replicas=3
kubectl -n shop expose deployment checkout --port=80
kubectl -n shop annotate deployment checkout kubernetes.io/change-cause="v2.13 good"
# "Bad release": nginx config that returns 500 for every request
kubectl -n shop create configmap bad-conf --from-literal=default.conf='server { listen 80; location / { return 500; } }'
kubectl -n shop patch deployment checkout --type=json -p='[
 {"op":"add","path":"/spec/template/spec/volumes","value":[{"name":"c","configMap":{"name":"bad-conf"}}]},
 {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts","value":[{"name":"c","mountPath":"/etc/nginx/conf.d"}]}]'
kubectl -n shop annotate deployment checkout kubernetes.io/change-cause="v2.14 bad" --overwrite
```

## How to find it

**First — impact and start time:** confirm error rate and the exact start minute (dashboard or logs).

```bash
kubectl -n shop run t --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'for i in $(seq 20); do wget -q -O /dev/null -T 2 http://checkout && echo 200 || echo ERR; done' | sort | uniq -c
```
```
     20 ERR
```

**Second — what changed?**

```bash
kubectl -n shop rollout history deployment/checkout
kubectl -n shop get rs -l app=checkout -o wide
```
```
REVISION  CHANGE-CAUSE
1         v2.13 good
2         v2.14 bad
NAME                  DESIRED  CURRENT  READY  AGE   IMAGES
checkout-6d5f9c7b8d   0        0        0      40m   nginx:1.27
checkout-7c9b4f6d5e   3        3        3      6m    nginx:1.27
```
- New ReplicaSet created 6 minutes ago → matches the error start. (Same image; the change is in the pod template — config.)

**Third — old vs new** (if both still exist during rollout): compare error rate by pod hash / version label; read new pods' logs.

```bash
kubectl -n shop logs -l app=checkout --tail=5 --prefix
kubectl -n shop diff -f checkout.yaml     # if you have the manifest
```

## How to fix it

### Immediate

```bash
kubectl -n shop rollout undo deployment/checkout          # back to revision 1
kubectl -n shop rollout status deployment/checkout
```
Helm: `helm rollback checkout <rev>`. **GitOps:** `git revert <commit>` and let Argo CD sync (otherwise it re-applies the bad version). Announce in the incident channel: "Rolled back checkout to v2.13 at 03:12."

Freeze further deploys of this service.

### Permanent

- Fix the bug + regression test.
- Canary with automated analysis on error rate and latency + auto-rollback (Argo Rollouts / Flagger) (topic 23).
- Config changes versioned with the release (Kustomize hash suffix) (topic 19).

## How to verify the fix

- Synthetic check: 20/20 OK. Dashboard: 5xx back to baseline; burn-rate alert clears.
- `kubectl get rs`: old ReplicaSet has 3 ready pods.
- **Git and cluster agree** (Argo CD "Synced") — the bad version can't come back on the next sync.
- Business metric (orders/min) recovered.

## Prevention

Canaries, deploy markers on dashboards, alerts tied to SLOs, deploy freezes during incidents, change failure rate tracking.

## Follow-up questions

1. **Q: How did this happen — root cause, not trigger?**
   A: Trigger: v2.14 deploy. Root cause: a config/code change that fails with production conditions, reaching 100% of traffic because there was no canary analysis.
2. **Q: How did you detect it? Why not earlier?**
   A: Burn-rate alert ~6 minutes after start. A canary comparing new vs old would have caught it at 5% traffic in a minute or two.
3. **Q: What did you check first, and why?**
   A: The exact start time vs deploy history, because a deploy right before a step change is the most likely cause and the fastest to undo.
4. **Q: How did you stop impact quickly?**
   A: Rolled back (in Git for GitOps), announced it, froze deploys.
5. **Q: Permanent fix?**
   A: Bug fix with test, canary + automated rollback, versioned config.
6. **Q: How did you confirm?**
   A: User-facing error rate back to baseline, synthetic checks green, Git and cluster in sync.
7. **Q: What if the rollback didn't fix it?**
   A: Then the deploy wasn't the (only) cause: check other changes in the window (config, flags, infra, dependencies, traffic) and DB migrations that the old version can't handle.
8. **Q: How would you prevent it?**
   A: Progressive delivery with automatic analysis, smaller releases, feature flags.
9. **Q: What would you do differently next time?**
   A: Make rollback a one-click, practised action and add deploy markers to every dashboard.
10. **Q: What if it happened at 10× traffic / during a sale?**
    A: Same action, faster: automated rollback matters more. Many teams freeze or require extended canaries during peak events.
