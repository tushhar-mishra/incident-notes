# 88 · Drill: One Availability Zone Is Gone

> **Level:** ★★★ advanced
>
> **You will practise:** losing an entire **AZ** (Availability Zone: one or more data centers in a cloud region): confirming it's the AZ (not your app), **evacuating** traffic and workloads, handling **stateful** services (databases, volumes pinned to the AZ), avoiding a **thundering herd** of rescheduling, and recognising the value of **static stability**.
>
> **You need:** topics 20, 21, 32, 35, 38, 39, 76. Helpful: [SD 23 · Availability](../../System-Design/notes/23-availability-and-the-nines.md), [SD 42 · Multi-region](../../System-Design/notes/42-multi-region-architecture.md).
>
> **Part 4 · On-call drills** · Previous: [87 · Cloud bill 5× overnight](87-drill-cloud-bill-5x.md) · Next: [89 · Monitoring is down too](89-drill-monitoring-down.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **14:02 — PAGES (many):** `KubeNodeNotReady ×14 (all us-east-1b)`, `TargetGroupUnhealthyHosts`, `RDS: orders-prod Multi-AZ failover started`, error rate 18%

**The idea.** Everything in one zone failed at once. Your design either **already has capacity** in the other zones (static stability) or must **launch it now** — which may be slow or impossible if the cloud control plane is struggling too.

```
confirm: all failures share zone=us-east-1b? provider health shows an AZ issue?
evacuate: remove the AZ from traffic (LB / DNS / mesh), cordon its nodes
capacity: do the other AZs have room? (N+1 by AZ) → scale up there, watch quotas
state: DB failover (Multi-AZ) done? volumes stuck in the dead AZ? StatefulSets pinned there?
recover: when the AZ returns, rebalance slowly (avoid herd)
```

**The one thing to know.** **Pre-provisioned spare capacity across AZs** turns this into a non-event; planning to "just scale up when needed" fails when everyone is scaling up at once.

> 📖 Read only **Symptoms**, then write what you check and do in the first ten minutes.

---

## Symptoms

- Nodes NotReady **only** in one zone; load balancer marks targets in that AZ unhealthy.
- Managed DB fails over to its standby in another AZ (brief write errors).
- Error rate spikes then partially recovers as the LB routes to healthy AZs — if there's enough capacity.
- Provider health dashboard (and AWS Health API for your account) shows an AZ event.

## Analogy

One of three kitchens in a restaurant chain loses power. If the other two always keep a few spare cooks and tables, customers barely notice. If they're already at full capacity, the queue explodes, and calling in extra cooks takes hours.

## How it happens

Power, cooling, network or storage failure in one AZ. Everything **zonal** fails: instances, EBS volumes, zonal subnets/NAT gateways, single-AZ databases. Regional services usually keep working, but sometimes control planes are degraded (topic 39).

### Lab setup (simulate)

```bash
# kind with labelled "zones"
cat > kind-az.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  labels: { topology.kubernetes.io/zone: az-a }
- role: worker
  labels: { topology.kubernetes.io/zone: az-b }
- role: worker
  labels: { topology.kubernetes.io/zone: az-c }
EOF
kind create cluster --name az --config kind-az.yaml
kubectl create deployment web --image=nginx:1.27 --replicas=6
kubectl patch deployment web -p '{"spec":{"template":{"spec":{"topologySpreadConstraints":[{"maxSkew":1,"topologyKey":"topology.kubernetes.io/zone","whenUnsatisfiable":"DoNotSchedule","labelSelector":{"matchLabels":{"app":"web"}}}]}}}}'
# "Lose" az-b: stop that node's container
docker stop $(docker ps --filter name=az-worker2 -q)
```

## How to find it

**First — is it the zone?**

```bash
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -o wide -l app=web
aws health describe-events --filter eventStatusCodes=open --region us-east-1   # (Business/Enterprise support)
```
```
NAME          STATUS     ZONE
az-worker     Ready      az-a
az-worker2    NotReady   az-b     ← whole zone
az-worker3    Ready      az-c
```

**Second — capacity in the remaining AZs:**

```bash
kubectl describe nodes | grep -A6 "Allocated resources"
kubectl get pods -A --field-selector=status.phase=Pending
```
- Pending pods with `didn't match pod topology spread constraints` (DoNotSchedule) or `Insufficient cpu` → not enough room or rules too strict.

**Third — state:** DB failover status; PVCs bound to volumes in the dead AZ (`kubectl get pv -L topology.kubernetes.io/zone`); StatefulSet pods stuck Terminating on dead nodes (topic 21).

## How to fix it

### Immediate

1. **Take the AZ out of traffic** (if health checks haven't already): disable the zone in the LB/target group or DNS weights; cordon its nodes: `kubectl cordon -l topology.kubernetes.io/zone=us-east-1b`.
2. **Add capacity** in healthy AZs (raise node group min/max; Karpenter will provision) — watch **quotas** and launch errors (topic 35).
3. **Relax spread constraints temporarily** if they block scheduling (`whenUnsatisfiable: ScheduleAnyway`) — accept imbalance to restore capacity.
4. **Stateful:** confirm DB failover completed and apps reconnected; for volumes trapped in the dead AZ, restore from snapshot in another AZ or wait for the AZ if data is safe; force-delete StatefulSet pods only after confirming the nodes are really down.
5. **Shed non-critical load** if capacity is short.

### Permanent

- **N+1 by AZ:** each AZ can lose one peer and the remaining ones carry 100% (e.g. 3 AZs each at ≤ 66% utilisation).
- Topology spread + PDBs; multi-AZ databases; storage that survives an AZ (replicated) where needed.
- Zonal NAT gateways per AZ; no cross-AZ single points.
- Game days: kill an AZ in staging regularly.

## How to verify the fix

- Error rate back in SLO with traffic only on healthy AZs; no Pending pods; DB writable.
- When the AZ recovers: **rebalance gradually** (uncordon, let the descheduler/new deploys spread pods) — not all at once.

## Prevention

Static stability (pre-provisioned spare capacity), quotas for AZ-loss scale-up, regular AZ evacuation drills, dependency review for zonal SPOFs.

## Follow-up questions

1. **Q: Root cause?**
   A: The provider's AZ failure was the trigger; our impact came from insufficient spare capacity in the remaining AZs and strict spread rules that blocked rescheduling.
2. **Q: How did you detect it? Why so many alerts?**
   A: Node, LB and DB alerts all fired at once; grouping by zone label showed the shared cause. Alertmanager grouping by zone would reduce noise.
3. **Q: What did you check first?**
   A: Whether failures shared a zone, then provider health — to know it's infrastructure, not our change.
4. **Q: How did you stop impact quickly?**
   A: Removed the AZ from traffic, added capacity elsewhere, relaxed spread constraints, shed non-critical load.
5. **Q: Permanent fix?**
   A: N+1 capacity per AZ, quota headroom, AZ-loss game days, zonal SPOF review.
6. **Q: How did you confirm?**
   A: SLO restored on two AZs; then gradual rebalance after recovery.
7. **Q: What if you couldn't launch new capacity (control plane impaired, quota)?**
   A: That's why static stability matters: shed load, prioritise critical services (PriorityClasses), and wait. Don't terminate anything you can't replace.
8. **Q: How would you prevent it?**
   A: Design for AZ loss as a normal event: spare capacity, multi-AZ data, regular drills.
9. **Q: What would you do differently?**
   A: Use `ScheduleAnyway` for spread on most services so they reschedule during AZ loss, keeping `DoNotSchedule` only where strict spreading truly matters.
10. **Q: What if it was a whole region?**
    A: Then it's a regional DR event (topic 39, SD 42): fail over to another region if prepared; otherwise degrade and communicate.
