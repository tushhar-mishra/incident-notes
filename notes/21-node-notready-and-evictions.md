# 21 · Node NotReady and Evictions

> **Level:** ★★ intermediate
>
> **You will learn:** how Kubernetes decides a node is **healthy** (kubelet **heartbeats** and **Lease** objects), what **NotReady** and **Unknown** mean, what happens to pods on a dead node (**taint-based eviction** after ~5 minutes), the **pressure conditions** (MemoryPressure, DiskPressure, PIDPressure) and **kubelet eviction thresholds**, the usual causes (kubelet down, container runtime down, disk full, network, expired kubelet certificate, OS update), and how to **cordon/drain** safely.
>
> **You need:** topics 01–20. Helpful: [K8s 08 · Node components](../../CKA-Certification-Course-2025/notes/08-node-components.md), [K8s 47 · Priority, preemption, eviction](../../CKA-Certification-Course-2025/notes/47-priority-preemption-eviction.md), [K8s 83 · Data plane troubleshooting](../../CKA-Certification-Course-2025/notes/83-data-plane-troubleshooting.md).
>
> **Part 2 · Recurring failures · B. Containers and Kubernetes** · Previous: [20 · Pending and ImagePullBackOff](20-pods-pending-and-imagepullbackoff.md) · Next: [22 · Control-plane overload](22-control-plane-overload.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Each node's **kubelet** sends a heartbeat to the control plane. If heartbeats stop for about 40–50 seconds (depends on version and settings), the node becomes **NotReady/Unknown**. After a further ~5 minutes (default toleration), its pods are **evicted** and recreated elsewhere. Separately, a **live** node under memory/disk/PID pressure **evicts its own pods** to protect itself.

```
kubelet heartbeat (Lease every ~10s) ──► API server
         stops ──► ~40–50s ──► node NotReady/Unknown (taint: node.kubernetes.io/unreachable)
                   ──► 300s toleration expires ──► pods marked for deletion, recreated elsewhere
```

**Picture it.** A security guard who must check in by radio every 10 seconds. No check-in for a while → the control room marks the post "unknown". After 5 more minutes → they send someone else to cover that post.

**The one thing to know.** One NotReady node is routine. **Many NotReady nodes at once** means something shared: network, a DaemonSet/CNI, an OS update, the control plane, or certificates. Look for the **common factor**.

**Try it now:** `kubectl get nodes` and `kubectl describe node <node> | grep -A10 Conditions`.

**You've got it if you can:** say why a node went NotReady using `describe node` and kubelet logs, and drain a node safely.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| `kubectl get nodes` | `STATUS: NotReady` (or `Unknown` conditions), `SchedulingDisabled` if cordoned |
| Pods | Stuck `Terminating` on the dead node; `Evicted` pods (pressure); replacements Pending if no room |
| Events | `NodeNotReady`, `NodeHasDiskPressure`, `EvictionThresholdMet`, `Evicted: The node was low on resource: memory` |
| Users | Reduced capacity; errors if many replicas were on the same node; stateful pods (StatefulSet) don't move automatically while the old pod may still exist |
| Cloud | Instance status checks failing; spot instance interruption |

---

## Analogy

The **radio check-in** (Core). Pressure eviction is like a lifeboat that's too full: the crew removes the least important cargo first so the boat doesn't sink.

---

## How it happens

### Why heartbeats stop (NotReady)

| Cause | Clue |
|---|---|
| **kubelet** stopped/crashed | `systemctl status kubelet` failed; kubelet logs |
| **Container runtime** (containerd) down | kubelet logs: `container runtime is down`, `PLEG is not healthy` |
| **Node out of resources** | Memory exhausted (kubelet itself OOM-killed), disk full, PID exhaustion |
| **Network** | Node can't reach API server (security group, routes, CNI broken, DNS for API endpoint) |
| **Expired kubelet client certificate** | kubelet logs: `x509: certificate has expired`; rotation disabled or failed (topic 27) |
| **OS / kernel problem** | Kernel panic, hung filesystem, an automatic OS update restarting network services (topics 63, 64) |
| **Instance gone** | Spot interruption, hardware failure, someone terminated it |

### Pressure conditions and kubelet eviction

The kubelet watches **eviction signals**. Default **hard** thresholds on Linux (configurable):

| Signal | Default hard threshold | Condition |
|---|---|---|
| `memory.available` | < 100Mi | MemoryPressure |
| `nodefs.available` | < 10% | DiskPressure |
| `imagefs.available` | < 15% | DiskPressure |
| `nodefs.inodesFree` | < 5% | DiskPressure |
| `pid.available` | (no default) | PIDPressure |

When a threshold is crossed, kubelet evicts pods, preferring those **using more than they requested**, then lower **priority** (BestEffort and over-request Burstable first). For DiskPressure it first removes dead containers and unused images.

### What happens to pods on a dead node

- The node gets taints `node.kubernetes.io/not-ready` or `node.kubernetes.io/unreachable` (NoExecute).
- Pods tolerate those taints for **300 seconds** by default (added automatically), then they're evicted.
- Deployment pods are recreated elsewhere. **StatefulSet** pods are **not** recreated until the old pod is confirmed gone (to avoid two copies writing the same volume) — you may need to delete the node object or force-delete the pod **after confirming the node is really dead**.

---

## How to find it

**First:** how many nodes, and what do they share?

```bash
kubectl get nodes -L topology.kubernetes.io/zone,node.kubernetes.io/instance-type,eks.amazonaws.com/nodegroup
```
```
NAME          STATUS     ROLES    AGE   VERSION   ZONE         INSTANCE-TYPE  NODEGROUP
ip-10-0-1-11  Ready      <none>   40d   v1.36.2   us-east-1a   m6i.xlarge     general
ip-10-0-2-12  NotReady   <none>   40d   v1.36.2   us-east-1b   m6i.xlarge     general
ip-10-0-2-19  NotReady   <none>   12d   v1.36.2   us-east-1b   m6i.xlarge     general
```
- Both NotReady nodes are in **us-east-1b** → suspect zone/network, not individual nodes.

**Second:** the node's conditions.

```bash
kubectl describe node ip-10-0-2-12 | sed -n '/Conditions/,/Addresses/p'
```
```
  Type             Status    LastHeartbeatTime   Reason              Message
  MemoryPressure   Unknown   ...                 NodeStatusUnknown   Kubelet stopped posting node status.
  DiskPressure     Unknown   ...                 NodeStatusUnknown   Kubelet stopped posting node status.
  Ready            Unknown   ...                 NodeStatusUnknown   Kubelet stopped posting node status.
```
- `Unknown` + "Kubelet stopped posting" → the control plane can't hear the kubelet (dead kubelet, dead node, or network). If instead `DiskPressure True` with Ready True → the node is alive but evicting.

**Third:** on the node itself (SSH, SSM, or cloud serial console):

```bash
systemctl status kubelet containerd
journalctl -u kubelet --since "30 min ago" | tail -30
df -h /var/lib/kubelet /var/lib/containerd; free -m
curl -sk https://<api-endpoint>/healthz     # can the node reach the API?
```
```
E1120 03:20:11 kubelet_node_status.go: "Error updating node status" err="Put https://...: dial tcp 10.0.0.5:443: i/o timeout"
```
- Network path to the API server is broken from this node.

Pressure evictions:
```bash
kubectl get events -A --field-selector reason=Evicted --sort-by=.lastTimestamp | tail
kubectl get pods -A --field-selector status.phase=Failed
```

---

## How to fix it

### Immediate

- **Protect capacity first:** make sure replacement pods can schedule (add nodes in healthy zones).
- **Cordon** the bad node so nothing new lands there: `kubectl cordon ip-10-0-2-12`.
- If the node is alive: restart the broken component (`systemctl restart containerd kubelet`), free disk (topic 13).
- If the node is dead: terminate it and let the node group / Karpenter replace it.
- **Drain** a live but unhealthy node safely:
  ```bash
  kubectl drain ip-10-0-2-12 --ignore-daemonsets --delete-emptydir-data --timeout=5m
  ```
  This respects **PodDisruptionBudgets** (PDBs), so it won't take down too many replicas at once.
- StatefulSet pod stuck on a dead node: confirm the instance is **really** stopped (cloud console), then `kubectl delete node <node>` (or force-delete the pod).

### Permanent

- Fix the shared cause (network change, OS update policy, certificate rotation, DaemonSet bug).
- Reserve resources for system daemons (`kube-reserved`, `system-reserved`) so kubelet and runtime survive pressure.
- Correct requests so kubelet doesn't need to evict; PDBs on every important workload; replicas spread across nodes and zones (topology spread).
- Node auto-repair (managed node groups, Karpenter node health).

---

## How to verify the fix

- All nodes `Ready`; no pressure conditions.
- No pods Pending or Terminating for long.
- Workloads at desired replicas, spread across zones.
- Error rate normal.

---

## Prevention

- Alerts: `kube_node_status_condition{condition="Ready",status="true"} == 0` for 5 min; more than N% of nodes NotReady (page); pressure conditions.
- Kubelet certificate rotation on (`rotateCertificates: true`), with expiry monitoring.
- Controlled OS updates: staged, one node pool at a time (topic 25).
- Regular node rotation (immutable nodes) so drift and old certificates don't accumulate.

---

## Follow-up questions

1. **Q: What was the root cause?**
   A: Example: an automatic OS package update restarted the network service on nodes in one node group, dropping routes needed by the CNI; kubelets couldn't reach the API server. Trigger: the update; root cause: unattended upgrades enabled on production nodes with no staging.

2. **Q: How did you detect it? Why not earlier?**
   A: NotReady alert and reduced capacity. Not earlier because updates ran on all nodes in the window at once; there was no canary node.

3. **Q: What did you check first?**
   A: What the NotReady nodes had in common (zone, node group, age, version), because many nodes failing together means a shared cause.

4. **Q: How did you stop impact?**
   A: Added capacity in healthy groups, cordoned the affected nodes, and replaced them from a known-good image.

5. **Q: Permanent fix?**
   A: Disabled unattended upgrades; OS updates now come only through new node images rolled out one pool at a time.

6. **Q: How did you confirm?**
   A: All nodes Ready, replicas spread correctly, and the next image rollout went pool by pool without impact.

7. **Q: What if the replacement nodes also went NotReady?**
   A: Then the cause is not the old nodes but something applied at boot (bad image, bootstrap script, CNI DaemonSet version, security group, API endpoint). Roll back the node image or DaemonSet, and check one new node in detail.

8. **Q: Why don't StatefulSet pods move automatically from a dead node?**
   A: Kubernetes can't be sure the old pod is gone; if it's still running behind a network partition, two copies could write the same volume and corrupt data. You confirm the node is dead and delete it (or force-delete the pod).

9. **Q: What if this happened during a deployment?**
   A: Capacity is already reduced by the rollout. Pause the rollout (`kubectl rollout pause`) so you're not fighting two problems, restore node capacity first, then resume.

10. **Q: What would you do differently?**
    A: PDBs and topology spread for all services, an alert on "% of nodes NotReady", and staged node updates.
