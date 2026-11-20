# 19 · CrashLoopBackOff

> **Level:** ★ foundation
>
> **You will learn:** what **CrashLoopBackOff** really means (it's a **state**, not a cause), how the **restart backoff** works, how to read **exit codes**, the **usual causes in order** (bad config/secret, missing dependency, wrong command, failing **liveness probe**, OOM, permissions, bad image), how to get the logs of the **previous** crashed container, and how to debug a container that dies too fast to `exec` into.
>
> **You need:** topics 01–18. Helpful: [K8s 19 · Pod lifecycle](../../CKA-Certification-Course-2025/notes/19-pod-lifecycle-restarts-and-termination.md), [K8s 21 · Debugging pods](../../CKA-Certification-Course-2025/notes/21-debugging-pods.md), [K8s 22 · Health probes](../../CKA-Certification-Course-2025/notes/22-health-probes.md), [Docker 12 · PID 1 and signals](../../Docker/notes/12-pid1-signals-and-shutdown.md).
>
> **Part 2 · Recurring failures · B. Containers and Kubernetes** · Previous: [18 · OOMKilled](18-oomkilled-in-kubernetes.md) · Next: [20 · Pods Pending and ImagePullBackOff](20-pods-pending-and-imagepullbackoff.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **CrashLoopBackOff** means: "this container keeps exiting, and kubelet is now **waiting longer and longer** before restarting it." It tells you **that** it crashes, not **why**. The why is in the **previous container's logs** and the **exit code**.

```
start → crash → wait 10s → start → crash → wait 20s → 40s → 80s → 160s → 300s (cap)
                                                    ↑ STATUS shows CrashLoopBackOff while waiting
```

**Picture it.** A car that stalls every time you start it. The dashboard light ("CrashLoopBackOff") only says "engine keeps stalling". You need the engine's error code (exit code) and what it was doing just before (logs).

**The one thing to know.** Run **`kubectl logs <pod> --previous`**. The current container may have no logs yet; the **previous** one has the error.

**Try it now:** `kubectl run crashy --image=busybox -- sh -c 'echo boom; exit 1'` and watch `kubectl get pod crashy -w`.

**You've got it if you can:** find the cause of a CrashLoopBackOff in under 5 minutes using describe, previous logs and exit codes.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| `kubectl get pods` | `STATUS: CrashLoopBackOff`, `READY 0/1`, `RESTARTS` increasing |
| Events | `Back-off restarting failed container <name> in pod <pod>` |
| Deployment | Rollout stuck: `Waiting for deployment "x" rollout to finish: 1 out of 3 new replicas have been updated...` |
| Users | If all replicas crash: 503s (Service has no ready endpoints). If only new pods crash during a rollout: often no impact (old pods still serve) — **this is why rolling updates with readiness probes are safe** |

---

## Analogy

The **stalling car** (Core). The growing wait between restarts is like waiting longer each time before trying again, so you don't flood the engine.

---

## How it happens

### The backoff

kubelet restarts failed containers (with `restartPolicy: Always`, the default for Deployments) with an exponential delay: by default starting at **10 s**, doubling, capped at **5 minutes**, and reset after the container runs successfully for **10 minutes**. (Newer Kubernetes versions add feature-gated options to tune this; defaults are as above.)

### Exit codes

| Exit code | Meaning | Usual cause |
|---|---|---|
| **0** | Exited successfully | A one-shot command in a Deployment (it's not a long-running server) |
| **1** | General application error | Exception at startup: bad config, can't connect, missing env var |
| **2** | Misuse of shell builtins / bad arguments | Wrong command/args |
| **126** | Command found but not executable | Missing execute permission, wrong architecture binary |
| **127** | Command not found | Wrong `command:`, typo, binary not in image, wrong `PATH` |
| **128 + N** | Killed by signal N | **137** = SIGKILL (OOM, or kubelet killing after a failed liveness probe/grace period); **143** = SIGTERM (asked to stop) |
| **139** | 128 + 11 = SIGSEGV | Segmentation fault: native crash, wrong CPU architecture/library |

### The usual causes, in the order to check

1. **Bad config, secret or env var** — the app reads config at startup and exits ("DATABASE_URL not set", "invalid YAML"). Most common after a ConfigMap/Secret change (drill 80).
2. **Dependency not reachable** — the app exits if it can't connect to the DB/cache at startup.
3. **Wrong command/args** — exit 127/126/2.
4. **Liveness probe kills it** — the app is healthy but slow to start; the probe fails, kubelet kills it (exit 137 without OOMKilled; Events say `Liveness probe failed`). Fix with a **startupProbe** or a longer delay.
5. **OOMKilled** — topic 18.
6. **Permissions** — non-root user can't write to a directory; read-only root filesystem; volume ownership.
7. **Bad image** — wrong architecture (`exec format error`), missing files, bad build.

---

## How to find it

**First:** describe the pod: exit code, reason, events.

```bash
kubectl describe pod api-6c9f7d8b4-x2v9q -n shop
```
```
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
...
Events:
  Warning  BackOff  2m (x14 over 6m)  kubelet  Back-off restarting failed container api in pod api-6c9f7d8b4-x2v9q
```
- `Exit Code: 1` → the app itself exited with an error. Logs will say why.
- If Events showed `Liveness probe failed: Get "http://10.0.3.7:8080/healthz": context deadline exceeded` → the probe is killing it.

**Second:** logs from the **previous** run.

```bash
kubectl logs api-6c9f7d8b4-x2v9q -n shop --previous
```
```
2026-11-20T03:14:02Z INFO starting api v3.2.0
2026-11-20T03:14:02Z ERROR config: required key "PAYMENTS_URL" missing
```
- Clear cause: a missing configuration key.

**Third:** what changed? Compare the new and old ReplicaSets.

```bash
kubectl rollout history deploy/api -n shop
kubectl diff -f api.yaml           # or compare Git commits / Argo CD diff
kubectl get rs -n shop -l app=api -o wide
```

**If it dies too fast to look inside:**

```bash
# Run a copy with the command replaced, so you can exec in
kubectl debug pod/api-6c9f7d8b4-x2v9q -n shop -it --copy-to=api-debug \
  --container=api -- sh
# Or add an ephemeral debug container to the running pod
kubectl debug -it api-6c9f7d8b4-x2v9q -n shop --image=busybox --target=api
```
Inside: check env (`env | sort`), mounted files (`ls -l /etc/config`), DNS (`nslookup postgres`), connectivity (`nc -zv postgres 5432`).

---

## How to fix it

### Immediate

- **Roll back** if a rollout caused it: `kubectl rollout undo deploy/api -n shop` (or revert in Git for GitOps).
- If only **new** pods crash and old ones serve traffic, there may be no user impact: pause and investigate calmly (`kubectl rollout pause`).
- If a ConfigMap/Secret change caused it, revert that change and restart the pods.
- If the **liveness probe** is killing a slow starter: add a `startupProbe` or increase `initialDelaySeconds`/`failureThreshold`.

### Permanent

- Validate config in CI (schema checks, required keys), and fail **readiness** (not crash) when a dependency is temporarily unavailable, with retries and backoff.
- Use **immutable, versioned ConfigMaps** (e.g. Kustomize `configMapGenerator` adds a hash suffix) so a config change is a **rollout** that can be rolled back ([K8s 64](../../CKA-Certification-Course-2025/notes/64-kustomize.md)).
- Correct probes: `startupProbe` for slow starts; liveness checks only "is the process stuck", not dependencies.
- Rollouts with `maxUnavailable: 0` + readiness probes so bad versions never take traffic.

---

## How to verify the fix

- Pods `Running`, `READY 1/1`, `RESTARTS` stable for > 10 minutes (backoff resets).
- `kubectl rollout status deploy/api` succeeds.
- Service has ready endpoints; error rate normal.

---

## Prevention

- Alert on restarts and on rollouts stuck longer than N minutes.
- Progressive delivery (canary) with automatic rollback (Argo Rollouts/Flagger).
- Config validation and smoke tests in the pipeline ([GHA 45 · build and test](../../GitHub-Actions-Basics-To-Production/notes/45-capstone-03-build-and-test.md), [Jenkins 41](../../Jenkins/notes/41-results-tests-reports.md)).

---

## Follow-up questions

1. **Q: What was the root cause, not the trigger?**
   A: The trigger was a deploy of v3.2.0. The root cause: the new version required a new config key that was added to the staging ConfigMap but not production's, and there was no config validation in the pipeline.

2. **Q: How did you detect it, and why not earlier?**
   A: The rollout stuck and the restart alert fired. Users weren't affected because the old pods kept serving. It wasn't caught earlier because staging had the key, so tests passed there.

3. **Q: What did you check first?**
   A: `describe` for exit code and events, then `logs --previous`, because CrashLoopBackOff itself says nothing about the cause.

4. **Q: How did you stop impact?**
   A: There was none for users; I paused/rolled back the rollout to keep old pods serving and free the pipeline.

5. **Q: Permanent fix?**
   A: Added the key through Git for prod, and added a CI check that every environment's config contains all required keys for the version being deployed.

6. **Q: How did you confirm?**
   A: Rollout completed, pods stable past the 10-minute backoff reset, error rate unchanged.

7. **Q: What if the logs are empty?**
   A: The process may die before logging (wrong command → exit 127, wrong architecture → `exec format error`, segfault 139). Check the exit code and Events; run the image locally or via `kubectl debug --copy-to` with a shell.

8. **Q: Exit code 137 but not OOMKilled — what is it?**
   A: Something sent SIGKILL: usually kubelet after a failed liveness probe and grace period. Check Events for `Liveness probe failed`.

9. **Q: What if all replicas crash at once in production?**
   A: That's a full outage. Roll back immediately; if a shared dependency (DB, config service) is the cause, the rollback won't help — fix the dependency, and make the app degrade instead of exiting.

10. **Q: What would you do differently?**
    A: Treat config as part of the release (versioned, validated, rolled out together), and add startup probes as a standard template.
