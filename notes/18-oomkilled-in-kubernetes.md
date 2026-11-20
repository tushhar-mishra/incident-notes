# 18 · OOMKilled in Kubernetes

> **Level:** ★ foundation
>
> **You will learn:** what **OOMKilled** and **exit code 137** mean, how **requests** and **limits** for memory work, the three **QoS** (Quality of Service) classes and who dies first, **container-limit OOM vs node-pressure eviction**, why **JVM/Node heap settings** must fit inside the limit, the exact commands to confirm it, and how to size memory properly.
>
> **You need:** topics 01–17 (especially 16). Helpful: [K8s 41 · Requests, limits, QoS, quotas](../../CKA-Certification-Course-2025/notes/41-requests-limits-qos-quotas.md), [K8s 47 · Priority, preemption, eviction](../../CKA-Certification-Course-2025/notes/47-priority-preemption-eviction.md), [Docker 31 · Resource limits](../../Docker/notes/31-resource-limits.md).
>
> **Part 2 · Recurring failures · B. Containers and Kubernetes** · Previous: [17 · Clock skew](17-clock-skew-and-ntp.md) · Next: [19 · CrashLoopBackOff](19-crashloopbackoff.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A container's **memory limit** is a hard ceiling enforced by the Linux cgroup. Go over it and the kernel kills the process (SIGKILL). Kubernetes shows `Reason: OOMKilled`, `Exit Code: 137`, and restarts the container.

```
resources:
  requests: { memory: 512Mi }   ← used by the scheduler to find a node with room
  limits:   { memory: 1Gi }     ← hard ceiling: exceed it → OOMKilled (137)
```

**Picture it.** A rented storage unit. The **request** is the size you booked (the company guarantees it's free). The **limit** is the door: you physically can't fit more inside. Try to push more in, and the manager throws out your boxes.

**The one thing to know.** `137 = 128 + 9` → the process got **SIGKILL** (signal 9). With `OOMKilled: true`, it hit its memory limit. Raising the limit fixes a real need; it only **delays** a leak.

**Try it now:** `kubectl get pods -A | grep -i oom` and `kubectl describe pod <pod> | grep -A5 "Last State"`.

**You've got it if you can:** distinguish "container OOMKilled" from "pod evicted for node memory pressure", and set requests/limits from real data.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| `kubectl get pods` | `STATUS: OOMKilled` briefly, then `CrashLoopBackOff` if repeated; `RESTARTS` climbing |
| `kubectl describe pod` | `Last State: Terminated  Reason: OOMKilled  Exit Code: 137` |
| Events | `Back-off restarting failed container` |
| Metrics | `container_memory_working_set_bytes` rising to the limit, then dropping (restart) |
| Users | Intermittent 502/503 during restarts; in-flight requests lost |
| Different but similar | `Evicted` pods with `The node was low on resource: memory` (node pressure, not the container limit) |

---

## Analogy

The **storage unit** (Core). **Eviction** is different: the whole building is overfull, so the manager removes tenants (starting with those who didn't book a guaranteed size).

---

## How it happens

### Requests, limits and QoS

| QoS class | When | OOM priority (`oom_score_adj`) |
|---|---|---|
| **Guaranteed** | Every container has CPU and memory requests **=** limits | Killed last (-997) |
| **Burstable** | At least one request or limit set, but not Guaranteed | In between (scaled by request) |
| **BestEffort** | No requests or limits at all | Killed first (1000) |

### Two different killers

| | **Container OOMKilled** | **Node-pressure eviction** |
|---|---|---|
| Who acts | Linux kernel (cgroup OOM killer) | kubelet |
| Why | The container went over **its own limit** | The **node** is low on memory (below `evictionHard`, e.g. `memory.available<100Mi`) |
| What you see | `OOMKilled`, exit 137, container restarts **in the same pod** | Pod `Evicted`, phase `Failed`; a **new** pod is created elsewhere (by the Deployment) |
| Usual fix | Fix leak / raise limit | Set requests correctly (nodes over-committed), add capacity |

### Common causes

- **Leak** (topic 16).
- **Limit too small** for the real working set (data grew, traffic grew).
- **Runtime heap bigger than the limit:** JVM `-Xmx2g` in a 1 Gi container; Node.js default heap; Python workers × memory each. The heap is not the only memory a process uses (threads, native buffers, metaspace).
- **Many processes in one container:** gunicorn/PHP-FPM workers each grow.
- **Page cache / tmpfs:** files written to `emptyDir: {medium: Memory}` count against memory.

---

## How to find it

**First:** confirm it's OOMKilled (not a crash with another exit code).

```bash
kubectl get pods -n shop
kubectl describe pod checkout-7d9f8b6c5d-2xkqp -n shop | sed -n '/Last State/,/Ready/p'
```
```
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Thu, 20 Nov 2026 03:01:12 +0000
      Finished:     Thu, 20 Nov 2026 03:12:44 +0000
    Ready:          True
```
- `OOMKilled` + `137` → over the container memory limit.
- Started → Finished = ~11 minutes of life → probably growth (leak or load), not instant.

All OOMKilled containers in the cluster:
```bash
kubectl get pods -A -o json | jq -r '.items[] | .metadata.namespace as $ns | .metadata.name as $p
  | .status.containerStatuses[]? | select(.lastState.terminated.reason=="OOMKilled")
  | "\($ns)/\($p) \(.name) restarts=\(.restartCount)"'
```

**Second:** limit vs real usage.

```bash
kubectl get pod <pod> -n shop -o jsonpath='{range .spec.containers[*]}{.name}{" "}{.resources}{"\n"}{end}'
kubectl top pod <pod> -n shop --containers
```
```
checkout {"limits":{"memory":"1Gi"},"requests":{"memory":"512Mi"}}
POD                         NAME       CPU(cores)   MEMORY(bytes)
checkout-7d9f8b6c5d-2xkqp   checkout   210m         968Mi
```
- 968Mi of 1Gi → close to the edge.

Look at the graph of `container_memory_working_set_bytes` over a day: **straight line up** = leak; **follows traffic** = undersized; **instant at startup** = config/heap sizing.

**Third:** was it the node instead?

```bash
kubectl get events -A --field-selector reason=Evicted --sort-by=.lastTimestamp | tail
kubectl describe node <node> | grep -A8 Conditions     # MemoryPressure?
kubectl describe node <node> | grep -A10 "Allocated resources"
```
If memory **limits** across pods add up to far more than the node has (overcommit), the node can run out even if no single pod is over its limit.

For JVM apps, check the heap flags:
```bash
kubectl exec <pod> -- sh -c 'java -XX:+PrintFlagsFinal -version 2>/dev/null | grep -E "MaxHeapSize|MaxRAMPercentage"'
```

---

## How to fix it

### Immediate

- **Roll back** if a new release started it.
- **Raise the limit** (and request) to stop the restarts while you investigate:
  ```bash
  kubectl set resources deploy/checkout -n shop --limits=memory=2Gi --requests=memory=1Gi
  ```
  ⚠️ This triggers a rolling restart, and in GitOps the next sync will revert it: change it in Git.
- Scale out to spread load if usage tracks traffic.

### Permanent

- Fix the leak (topic 16).
- Size from data: requests ≈ typical p95 usage, limit with headroom above peak. **VPA** (Vertical Pod Autoscaler) in recommendation mode gives suggestions ([K8s 49](../../CKA-Certification-Course-2025/notes/49-vpa-inplace-resize-node-autoscaling.md)).
- Make runtimes container-aware: JVM `-XX:MaxRAMPercentage=75`; Node `--max-old-space-size` below the limit; fewer worker processes.
- Critical services: memory request = limit (Guaranteed for memory).
- Avoid node overcommit: ResourceQuotas and LimitRanges so every pod has requests.

---

## How to verify the fix

- `RESTARTS` stops increasing; no new `OOMKilled` in `lastState`.
- Memory working set stays well below the limit (e.g. < 80%) over a full traffic cycle.
- No `Evicted` pods; node has no `MemoryPressure`.
- Error rate back in SLO.

---

## Prevention

- Alert on `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` and on restarts (`increase(kube_pod_container_status_restarts_total[1h]) > 3`).
- Alert when working set > 90% of limit for 15 min.
- LimitRange defaults so nothing runs without requests/limits.
- Load and soak tests with production limits.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: the JVM had `-Xmx` set to 1 GiB inside a 1 GiB container, so heap plus metaspace, thread stacks and native buffers exceeded the limit under load. The trigger was peak traffic; the root cause was runtime sizing that ignored non-heap memory.

2. **Q: How did you detect it? Why not earlier?**
   A: 502s during restarts and the restart-count alert. Not earlier because we had no "working set near limit" alert.

3. **Q: What did you check first?**
   A: `kubectl describe pod` for `Last State` reason and exit code, to confirm OOMKilled (and not a crash or eviction), since each has a different fix.

4. **Q: How did you stop impact?**
   A: Raised the limit through Git (so GitOps didn't revert it) and scaled out.

5. **Q: Permanent fix?**
   A: `MaxRAMPercentage=75` instead of a fixed `-Xmx`, requests/limits sized from two weeks of usage, Guaranteed memory for the service, and alerts.

6. **Q: How did you confirm?**
   A: No restarts for a week including peak days; working set under 75% of the limit.

7. **Q: What if raising the limit didn't help?**
   A: If it still OOMs at the new limit after a while, it's a leak: take a heap dump from one pod before it dies, compare snapshots, and roll back to the last good version meanwhile.

8. **Q: OOMKilled vs Evicted: how do you tell, and why does it matter?**
   A: OOMKilled = the container exceeded its own limit (kernel kill, same pod restarts). Evicted = the node ran low and kubelet removed the pod (pod Failed, replaced elsewhere). The first is about this container's sizing or leak; the second is about node capacity and requests across all pods.

9. **Q: What if it happened during a deploy?**
   A: New pods might use more memory at startup (caches warming, JIT). Check if old pods are fine and only new ones die: that points at the release or its config. Roll back.

10. **Q: What would you do differently?**
    A: Require memory requests/limits and runtime heap settings in the service template, plus working-set alerts, before the service goes live.
