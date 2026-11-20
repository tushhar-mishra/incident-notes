# 80 · Drill: CrashLoopBackOff After a ConfigMap Change

> **Level:** ★★ intermediate
>
> **You will practise:** treating **configuration as a change**: pods crash after someone edited a ConfigMap (no image change), reading **previous** logs, finding the config diff, rolling config back, and making config changes **versioned and rollback-able**.
>
> **You need:** topics 19, 24, 76. Helpful: [K8s 34 · ConfigMaps](../../CKA-Certification-Course-2025/notes/34-configmaps.md), [K8s 64 · Kustomize](../../CKA-Certification-Course-2025/notes/64-kustomize.md).
>
> **Part 4 · On-call drills** · Previous: [79 · Latency up, nothing deployed](79-drill-latency-no-deploy.md) · Next: [81 · "certificate has expired"](81-drill-certificate-expired.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **04:15 — PAGE:** `KubeDeploymentReplicasMismatch — payments/api 0/4 ready; pods CrashLoopBackOff`

**The idea.** No new image, but a **ConfigMap** was edited and pods restarted (manually, by a reloader, or by a node drain). New pods read the bad config and exit.

```
kubectl describe pod → exit code 1 → kubectl logs --previous → "invalid value for TIMEOUT_MS: '30s'"
→ who changed the ConfigMap, when? (Git history / audit log / Argo CD diff)
→ revert the config (in Git) → restart pods → verify → make config versioned (hash-suffixed)
```

**The one thing to know.** A plain ConfigMap edit **isn't a rollout**: `kubectl rollout undo` won't revert it. Revert the **config itself**, then restart.

> 📖 Read only **Symptoms**, then write your first three checks.

---

## Symptoms

- `0/4 ready`, `CrashLoopBackOff`, restarts climbing; Service has no endpoints → 503s for payments API.
- `kubectl rollout history` shows **no new revision** (image unchanged).
- Some pods may still be running the old config if they haven't restarted yet (mixed state).

## Analogy

The recipe card in the kitchen was rewritten overnight. Every new cook who starts a shift reads the wrong card and burns the dish; cooks who started yesterday are still fine — until they go on break.

## How it happens

- ConfigMaps are often edited directly (`kubectl edit`) or in Git and synced.
- Pods read env vars/files **at startup**; a reloader (e.g. Stakater Reloader) or any restart (eviction, drain, crash) picks up the bad config.
- Because the Deployment's pod template didn't change, there's **no rollout revision** to undo.

### Lab setup (break it)

```bash
kubectl -n shop create configmap api-config --from-literal=TIMEOUT_MS=3000
kubectl -n shop apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 4
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
      - name: api
        image: busybox:1.36
        envFrom: [{ configMapRef: { name: api-config } }]
        command: ["sh","-c","case \"$TIMEOUT_MS\" in ''|*[!0-9]*) echo \"FATAL invalid value for TIMEOUT_MS: '$TIMEOUT_MS' (expected integer ms)\"; exit 1;; esac; echo started; sleep 3600"]
EOF
kubectl -n shop rollout status deploy/api
# The "bad change" at night:
kubectl -n shop patch configmap api-config -p '{"data":{"TIMEOUT_MS":"30s"}}'
kubectl -n shop rollout restart deploy/api      # simulates a reloader / node drain
```

## How to find it

**First — why are pods dying?**

```bash
kubectl -n shop get pods -l app=api
kubectl -n shop describe pod <pod> | grep -A4 "Last State"
kubectl -n shop logs <pod> --previous
```
```
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
FATAL invalid value for TIMEOUT_MS: '30s' (expected integer ms)
```

**Second — what changed?** No image change:

```bash
kubectl -n shop rollout history deploy/api        # only restart annotations, same template otherwise
kubectl -n shop get configmap api-config -o yaml | grep -A3 "^data"
kubectl -n shop get configmap api-config -o jsonpath='{.metadata.managedFields[*].time}'; echo
```
Then: `git log -p -- k8s/payments/api-config.yaml`, Argo CD history/diff, or the Kubernetes audit log (`verb=patch`, `resource=configmaps`).

**Third — blast radius:** which other Deployments use this ConfigMap?

```bash
kubectl -n shop get deploy -o json | jq -r '.items[] | select(.. | .configMapRef?.name? == "api-config") | .metadata.name'
```

## How to fix it

### Immediate

```bash
# Revert the config (in Git for GitOps; directly only if that's your source of truth)
kubectl -n shop patch configmap api-config -p '{"data":{"TIMEOUT_MS":"3000"}}'
kubectl -n shop rollout restart deploy/api
kubectl -n shop rollout status deploy/api
```
Announce: "Reverted api-config TIMEOUT_MS to 3000 and restarted api at 04:24."

### Permanent

- **Versioned config:** Kustomize `configMapGenerator` (name gets a content hash), so a config change **creates a new rollout** that can be rolled back, and old pods keep the old ConfigMap.
- **Validate config in CI** (schema, types) before it reaches the cluster.
- Apps should **fail readiness, not crash**, on invalid optional config — or crash early in a canary only.
- No direct `kubectl edit` in production; changes via Git with review.

## How to verify the fix

- `4/4 ready`, restarts stable > 10 minutes, Service endpoints present, 503s gone.
- Git = cluster (no drift that will re-apply the bad value).

## Prevention

Config in Git with schema validation, hash-suffixed ConfigMaps, alerts on CrashLoopBackOff and replica mismatch, audit logging on ConfigMap changes.

## Follow-up questions

1. **Q: Root cause?**
   A: An unvalidated config value (`30s` instead of milliseconds) applied directly to a shared ConfigMap, picked up by all pods on restart. Contributing: config changes aren't rollouts, no validation, no canary for config.
2. **Q: How did you detect it? Why not earlier?**
   A: Replica mismatch / crashloop alert after pods restarted. Validation in CI would have stopped it before it reached the cluster.
3. **Q: What did you check first?**
   A: `describe` + `logs --previous` for the exit reason, which named the bad key directly.
4. **Q: How did you stop impact?**
   A: Reverted the value and restarted the Deployment.
5. **Q: Permanent fix?**
   A: Hash-suffixed ConfigMaps via Kustomize, CI validation, GitOps-only changes.
6. **Q: How did you confirm?**
   A: All replicas ready and stable, endpoints back, errors gone, Git and cluster in sync.
7. **Q: What if reverting the ConfigMap didn't help?**
   A: Pods may read config from elsewhere (Secret, mounted file cached, external config service) or another change landed at the same time; check all config sources the app reads and the Argo CD diff.
8. **Q: How would you prevent it?**
   A: Treat config exactly like code: reviewed, validated, versioned, rolled out progressively.
9. **Q: Why didn't `kubectl rollout undo` work?**
   A: The Deployment's pod template didn't change, so there's no previous revision representing the old config. With hash-suffixed ConfigMaps, the template references a new name, and undo works.
10. **Q: What if this happened during a node drain at peak?**
    A: Pods restart gradually, so impact grows as nodes drain — pause the drain (`kubectl uncordon` / stop the upgrade), revert config, then resume.
