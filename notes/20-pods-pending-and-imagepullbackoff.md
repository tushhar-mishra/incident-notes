# 20 · Pods Pending and ImagePullBackOff

> **Level:** ★ foundation
>
> **You will learn:** the two most common reasons new pods never start: **Pending** (the **scheduler** can't find a node: not enough CPU/memory, taints, affinity, PVC zone mismatch, quota) and **ImagePullBackOff / ErrImagePull** (the **kubelet** can't pull the image: wrong tag, no credentials, registry rate limits, registry down, wrong architecture), how to read the scheduler's message, and why these hit hardest **during scale-up and incidents**.
>
> **You need:** topics 01–19. Helpful: [K8s 20 · Images and pull policy](../../CKA-Certification-Course-2025/notes/20-images-and-pull-policy.md), [K8s 42 · Scheduler](../../CKA-Certification-Course-2025/notes/42-scheduler-manual-scheduling-static-pods.md), [K8s 43 · Taints and tolerations](../../CKA-Certification-Course-2025/notes/43-taints-and-tolerations.md), [K8s 62 · Private registries](../../CKA-Certification-Course-2025/notes/62-private-registries.md), [Docker 22 · Registries](../../Docker/notes/22-tagging-pushing-registries.md).
>
> **Part 2 · Recurring failures · B. Containers and Kubernetes** · Previous: [19 · CrashLoopBackOff](19-crashloopbackoff.md) · Next: [21 · Node NotReady and evictions](21-node-notready-and-evictions.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A pod goes through two gates before its app even starts:

```
Gate 1: SCHEDULER  "Which node has room and is allowed?"   ── fails → Pending
Gate 2: KUBELET    "Can I pull the image on this node?"     ── fails → ErrImagePull → ImagePullBackOff
then:   container starts                                   ── fails → CrashLoopBackOff (topic 19)
```

**Picture it.** Checking into a hotel. Pending = no room available that matches your booking (size, floor, pet-friendly). ImagePullBackOff = you have a room, but your luggage (the image) is stuck at the airport.

**The one thing to know.** The answer is almost always in **`kubectl describe pod` → Events**. The scheduler writes exactly why: `0/12 nodes are available: 4 Insufficient memory, 8 node(s) had untolerated taint…`.

**Try it now:** `kubectl run nope --image=nginx:doesnotexist` and read its Events.

**You've got it if you can:** read a `FailedScheduling` message and say what to change.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Pods | `Pending` for minutes; or `ErrImagePull` → `ImagePullBackOff` |
| Events | `FailedScheduling`, `Failed to pull image`, `toomanyrequests`, `401 Unauthorized`, `manifest unknown` |
| Autoscaling | HPA scales up but new pods stay Pending → no extra capacity during a traffic spike |
| Rollouts | Stuck; new pods never become Ready |
| Incidents | During a node/AZ failure, replacement pods can't start (no room, or every node pulls images at once and hits a rate limit) |

---

## Analogy

The **hotel** (Core).

---

## How it happens

### Pending: the scheduler's reasons

| Message fragment | Meaning | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | No node has enough **unrequested** capacity for the pod's **requests** | Add nodes (cluster autoscaler/Karpenter), lower inflated requests, remove waste |
| `node(s) had untolerated taint {key: value}` | Nodes are tainted (e.g. GPU, dedicated, control-plane, `node.kubernetes.io/unreachable`) | Add a toleration if intended, or target other nodes |
| `node(s) didn't match Pod's node affinity/selector` | `nodeSelector`/affinity labels don't match any node | Fix the label or the selector |
| `node(s) had volume node affinity conflict` | The PVC's disk is in **zone A**; no schedulable node in zone A | Add capacity in that zone; use `WaitForFirstConsumer` StorageClasses |
| `pod has unbound immediate PersistentVolumeClaims` | PVC not bound (no StorageClass, provisioner failing) | Fix storage ([K8s 38](../../CKA-Certification-Course-2025/notes/38-persistent-storage-pv-pvc.md), [K8s 39](../../CKA-Certification-Course-2025/notes/39-storageclasses-dynamic-provisioning.md)) |
| `Too many pods` | Node at max pods (often limited by IP addresses per node in cloud CNIs) | More/bigger nodes, prefix delegation |
| `didn't match pod topology spread constraints` / anti-affinity | Spreading rules can't be satisfied | Relax `whenUnsatisfiable`, add nodes in more zones |
| (no event; pod not created at all) | **ResourceQuota** exceeded — the ReplicaSet can't **create** the pod | `kubectl describe rs` shows `exceeded quota` |

### ImagePullBackOff: the kubelet's reasons

| Error text | Meaning | Fix |
|---|---|---|
| `manifest unknown` / `not found` | Tag doesn't exist (typo, CI didn't push, tag deleted by retention policy) | Fix the tag; use **digests** (`@sha256:…`) for prod |
| `401 Unauthorized` / `pull access denied` | No or wrong credentials | `imagePullSecrets`, node IAM role for ECR, refresh expired token |
| `toomanyrequests: You have reached your pull rate limit` | Docker Hub (or other registry) rate limit | Authenticate, use a pull-through cache/mirror, copy images to your own registry |
| `dial tcp … i/o timeout` / `no such host` | Network/DNS to the registry broken; registry down | Check egress/NAT, DNS, registry status |
| `no match for platform in manifest` / `exec format error` (at start) | Image built for the wrong CPU architecture (amd64 vs arm64) | Multi-arch builds ([Docker 21 · buildx](../../Docker/notes/21-buildx-and-multi-platform.md)) |

Docker Hub limits pulls for unauthenticated and free accounts. The numbers have changed several times; check Docker's current "usage and limits" page. Unauthenticated limits are per IP address, so a whole cluster behind one NAT gateway shares one limit.

---

## How to find it

**First:** describe the pod and read Events.

```bash
kubectl describe pod web-5d8f-abcde -n shop | sed -n '/Events/,$p'
```
```
Events:
  Warning  FailedScheduling  30s  default-scheduler  0/12 nodes are available:
    4 Insufficient memory, 6 node(s) had untolerated taint {dedicated: gpu},
    2 node(s) had volume node affinity conflict.
    preemption: 0/12 nodes are available: 12 No preemption victims found for incoming pod.
```
- 4 nodes: not enough **requested** memory free.
- 6 nodes: GPU-only (tainted).
- 2 nodes: wrong zone for the pod's volume.
- → Real capacity for this pod = **zero**. Add normal nodes in the volume's zone, or reduce the request.

```
  Warning  Failed     12s  kubelet  Failed to pull image "registry.example.com/web:v3.4.1":
           rpc error: ... manifest unknown
  Normal   BackOff    10s  kubelet  Back-off pulling image "registry.example.com/web:v3.4.1"
```
- Tag `v3.4.1` doesn't exist in the registry.

**Second:** capacity view.

```bash
kubectl describe nodes | grep -A6 "Allocated resources"
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A --field-selector=status.phase=Pending
```
```
  Resource           Requests      Limits
  cpu                3810m (97%)   7 (178%)
  memory             14Gi (93%)    20Gi (133%)
```
- **Requests** at 93–97% → the scheduler sees the node as full, even if **actual** use is lower. Inflated requests waste capacity.

**Third:** registry checks.

```bash
crane manifest registry.example.com/web:v3.4.1   # or: docker manifest inspect / skopeo inspect
kubectl get secret regcred -n shop -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.auths | keys'
kubectl get sa default -n shop -o yaml | grep -A2 imagePullSecrets
```

Is the autoscaler working?
```bash
kubectl -n kube-system logs deploy/cluster-autoscaler --tail=50   # or Karpenter controller logs
```

---

## How to fix it

### Immediate

- **Pending (capacity):** add nodes (raise node group max / let autoscaler work), or temporarily scale down non-critical workloads; use **PriorityClasses** so critical pods preempt low-priority ones ([K8s 47](../../CKA-Certification-Course-2025/notes/47-priority-preemption-eviction.md)).
- **Pending (rules):** fix selector/toleration in Git and roll out.
- **Image tag missing:** roll back to the last good tag; re-run the build/push.
- **Auth:** recreate the pull secret / fix the node role; ECR tokens expire after 12 hours, so static secrets made from them break.
- **Rate limit:** authenticate pulls immediately; switch the image to your mirror.

### Permanent

- Mirror all third-party images into your own registry (ECR pull-through cache, Harbor proxy cache, Artifactory).
- Deploy by **digest**, and only after CI confirms the push succeeded.
- Right-size requests; cluster autoscaler/Karpenter with sensible max limits; spare capacity (placeholder low-priority "balloon" pods) for fast scale-up.
- Zone-aware storage (`WaitForFirstConsumer`) and multi-zone node groups.
- Pre-pull critical images on nodes (DaemonSet or node image).

---

## How to verify the fix

- No pods in `Pending`/`ImagePullBackOff`: `kubectl get pods -A | grep -Ev "Running|Completed"`.
- New pods start in seconds; rollout completes; HPA scale-ups produce Ready pods.
- Autoscaler logs show successful scale-ups.

---

## Prevention

- Alert on pods Pending > 5–10 minutes (`kube_pod_status_phase{phase="Pending"}`) and on `ImagePullBackOff` reasons (`kube_pod_container_status_waiting_reason`).
- Dashboard: requested vs allocatable per node group; headroom.
- CI checks the image exists for the exact digest before deploying.
- Game day: kill an AZ and watch whether replacements can schedule and pull (topic 88).

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: during a traffic spike, HPA scaled out but pods stayed Pending because the node group was at its max size, and requests were inflated 3× above real use. Root cause: capacity limits and request sizing never reviewed; contributing: no Pending alert.

2. **Q: How did you detect it, and why not earlier?**
   A: Latency rose while HPA showed desired replicas higher than ready replicas. No Pending-pods alert existed.

3. **Q: What did you check first?**
   A: `kubectl describe pod` Events, because the scheduler or kubelet states the exact reason.

4. **Q: How did you stop the impact?**
   A: Raised the node group max, scaled down a non-critical batch workload, and let PriorityClasses favour the API.

5. **Q: Permanent fix?**
   A: Right-sized requests from VPA recommendations, Karpenter with headroom, Pending alerts, mirrored images.

6. **Q: How did you confirm?**
   A: Pending count zero; next spike scaled smoothly; time from HPA decision to Ready pod under 2 minutes.

7. **Q: What if adding nodes didn't fix Pending?**
   A: Then it isn't capacity: re-read the message — taints, affinity, zone conflict for the volume, max pods per node (IP exhaustion), or quota. New nodes in the wrong zone or without the needed label don't help.

8. **Q: Why do image pulls fail more during big incidents?**
   A: When many nodes are replaced at once, every new node pulls every image at the same time: registry rate limits and NAT bandwidth get hit exactly when you need fast recovery. Mirrors, caches and pre-pulled images avoid it.

9. **Q: Why use digests instead of tags?**
   A: Tags are mutable and can be deleted or overwritten; a digest identifies exact content, so what you tested is what runs, and a rollback really goes back.

10. **Q: What would you do differently?**
    A: Alert on Pending, keep capacity headroom, and own every image in our registry.
