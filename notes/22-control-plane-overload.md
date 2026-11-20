# 22 · Control-Plane Overload

> **Level:** ★★★ advanced
>
> **You will learn:** what happens when the Kubernetes **control plane** (API server + etcd) is overloaded: why **expensive LIST requests**, **watch storms**, too many **controllers/operators**, **DaemonSets on thousands of nodes** and **huge objects** can bring it down, the **etcd database size limit** (`mvcc: database space exceeded`), **API Priority and Fairness**, why **existing pods keep running** but **nothing can change**, why DNS and other data-plane pieces can still fail, and how to get back in when `kubectl` itself times out.
>
> **You need:** topics 01–21. Helpful: [K8s 07 · Control plane components](../../CKA-Certification-Course-2025/notes/07-control-plane-components.md), [K8s 14 · The Kubernetes API](../../CKA-Certification-Course-2025/notes/14-the-kubernetes-api.md), [K8s 77 · etcd backup/restore](../../CKA-Certification-Course-2025/notes/77-etcd-backup-restore-and-dr.md), [K8s 82 · Control plane troubleshooting](../../CKA-Certification-Course-2025/notes/82-control-plane-troubleshooting.md).
>
> **Part 2 · Recurring failures · B. Containers and Kubernetes** · Previous: [21 · Node NotReady](21-node-notready-and-evictions.md) · Next: [23 · Bad deploys](23-bad-deploys.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Everything in Kubernetes talks to the **API server**, which stores state in **etcd**. If too many clients ask for too much at once, the API server runs out of CPU/memory, etcd slows down, and the cluster becomes **unchangeable**: no deploys, no scaling, no new pods, and sometimes no DNS updates.

```
                 ┌───────────────┐       ┌──────┐
 kubectl, CI ───►│               │──────►│ etcd │
 controllers ───►│  API server   │       └──────┘
 kubelets (×N) ─►│ (CPU, memory) │
 DaemonSet that  │  overloaded   │◄── e.g. 5,000 pods each LISTing all pods every 10s
 LISTs pods (×N)►└───────────────┘
```

**Picture it.** A city hall with one front desk. Normally fine. Then 5,000 people arrive at once, each asking for a full copy of the city's records. Nobody else, not even the fire brigade, can get a permit.

**The one thing to know.** **Running pods usually keep running** when the control plane is down (the data plane doesn't need the API server for every packet). What breaks is anything that needs a **change** or a **fresh lookup** through the API: deploys, scaling, new pods, and (in OpenAI's December 2024 outage, topic 65) **DNS-based service discovery** once caches expired.

**Try it now:** `kubectl get --raw /metrics | grep apiserver_request_total | head` on a lab cluster.

**You've got it if you can:** list three things that can overload a control plane and how to get back control when `kubectl` times out.

> 📖 **That's the core.** Everything below is the **full reference**. Real case: [65 · OpenAI, December 2024](65-openai-kubernetes-2024.md).

---

## Symptoms

| Where | What you see |
|---|---|
| `kubectl` | Very slow or `Error from server (Timeout)`, `the server was unable to return a response in the time allotted`, `etcdserver: request timed out`, `429 Too Many Requests` |
| CI/CD | Deploys hang or fail; Argo CD shows sync errors |
| Controllers | HPA doesn't scale; new pods aren't created; endpoints don't update |
| Nodes | Kubelets can't update status → nodes may go NotReady (topic 21) |
| DNS | CoreDNS can't refresh endpoints → stale or failing service discovery as caches expire |
| etcd | `etcdserver: mvcc: database space exceeded` → cluster becomes read-only for writes; high `etcd_disk_wal_fsync_duration_seconds` |
| Managed (EKS/GKE/AKS) | Provider may scale the control plane, but it can still be overwhelmed or throttle you |

---

## Analogy

The **city hall** (Core). **API Priority and Fairness** is the city hall splitting the queue: emergency services get their own window, so a crowd of tourists can't block them.

---

## How it happens

| Cause | Why it's expensive |
|---|---|
| **Unpaginated LIST of big resources** (all pods, all secrets, all events) | Each LIST builds a huge response in memory; many at once → API server OOM/CPU |
| **Clients on every node** (a DaemonSet agent) that LIST/WATCH cluster-wide | Load grows with **node count × resource count**; fine at 50 nodes, fatal at 5,000 |
| **Relist storms** | After an API server restart, every watcher reconnects and re-LISTs at once (thundering herd, topic 37) |
| **Buggy controller/operator** | A hot loop updating objects; constant conflicts; writes amplify in etcd |
| **Huge objects / too many objects** | Thousands of ConfigMaps, giant CRDs, millions of Events → etcd grows |
| **etcd disk too slow** | etcd needs low-latency fsync; slow disks (or a noisy neighbour) slow every write |
| **etcd size quota** | Default backend quota is **2 GiB** (`--quota-backend-bytes`); when exceeded etcd raises a **NOSPACE alarm** and rejects writes until you compact, defragment and disarm the alarm |

### What keeps working, and what doesn't

| Keeps working (usually) | Breaks |
|---|---|
| Containers already running | Deploys, scaling, new pods, restarts that need rescheduling |
| Traffic through existing kube-proxy/CNI rules | Updates to Service endpoints (new/removed pods not reflected) |
| DNS answers **from cache** | DNS for names whose cache expired, if CoreDNS can't sync |
| | kubelet status updates → NotReady → evictions when it recovers |

### API Priority and Fairness (APF)

Since Kubernetes 1.29 (GA), the API server classifies requests into **priority levels** and **flows** (FlowSchemas), queues them fairly and rejects excess with **429**. System traffic (leader election, node heartbeats) gets protected priority. You can add FlowSchemas to cap a noisy client.

---

## How to find it

**First:** is the API server healthy and who is calling it?

```bash
kubectl get --raw='/readyz?verbose' | tail -5
kubectl get --raw /metrics | grep -E 'apiserver_request_total' | sort -t' ' -k2 -n | tail
```
Better, in Prometheus/Grafana:
```
topk(10, sum by (resource, verb, client) (rate(apiserver_request_total[5m])))
sum by (resource, verb) (rate(apiserver_request_duration_seconds_count{verb="LIST"}[5m]))
apiserver_current_inflight_requests
apiserver_flowcontrol_rejected_requests_total
```
- A spike in `LIST pods` from one `client`/user agent (e.g. `telemetry-agent/1.0`) is your culprit.

Managed clusters: check the **audit logs** (EKS control plane logs in CloudWatch, GKE Cloud Logging) for top user agents and verbs.

**Second:** etcd health.

```bash
# on a kubeadm control-plane node
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint status -w table
etcdctl ... alarm list
```
```
+------------------------+------------------+---------+---------+-----------+
|        ENDPOINT        |        ID        | VERSION | DB SIZE | IS LEADER |
+------------------------+------------------+---------+---------+-----------+
| https://127.0.0.1:2379 | 8e9e05c52164694d |  3.5.x  |  2.1 GB |      true |
+------------------------+------------------+---------+---------+-----------+
memberID:10276657743932975437 alarm:NOSPACE
```
- `DB SIZE 2.1 GB` and `alarm:NOSPACE` → etcd hit its quota; writes rejected.

**Third:** what changed? A new DaemonSet/operator, an upgrade, a big batch of objects created, node count growth.

```bash
kubectl get ds -A --sort-by=.metadata.creationTimestamp | tail
kubectl get events -A --sort-by=.lastTimestamp | tail     # may itself time out
```

---

## How to fix it

### Immediate: reduce load, regain control

1. **Stop the noisy client.** If `kubectl` still works (even slowly): `kubectl delete ds telemetry-agent -n monitoring` or scale its deployment to 0. If it doesn't:
   - Block the client at the network level (OpenAI's write-up describes blocking network access to the Kubernetes admin API to reduce load).
   - Use a **break-glass** path: an admin endpoint/credential with protected priority, or ask the managed provider to help.
   - Add an APF FlowSchema that puts the noisy client in a tiny priority level.
2. **Scale the API servers** (self-managed) or let the provider scale them.
3. **etcd NOSPACE:** compact old revisions, defragment, then disarm:
   ```bash
   etcdctl compact <current-revision>
   etcdctl defrag --cluster
   etcdctl alarm disarm
   ```
   ⚠️ Take an **etcd snapshot** first. Defrag blocks the member briefly; do members one at a time.
4. **Pause automation** that makes changes (CI deploys, autoscalers) until stable.

### Permanent

- Clients must use **informers/watch caches** and **pagination**, and list with label/field selectors, not cluster-wide LISTs on every node.
- **Load-test** new cluster-wide agents at the real node count before rollout; roll out DaemonSets **gradually** (a few nodes, then more) while watching API server metrics.
- APF FlowSchemas for known heavy clients.
- Keep etcd small: limit Events TTL, clean old objects, avoid huge CRDs; fast dedicated disks; monitor DB size.
- Split very large fleets into **more clusters** (smaller blast radius).
- Prepare a **break-glass** access path that works when the API is overloaded.

---

## How to verify the fix

- API server latency (`apiserver_request_duration_seconds` p99) and in-flight requests back to baseline; no 429/5xx spikes.
- `kubectl` responsive; a test deploy succeeds; HPA scales; endpoints update.
- etcd DB size below quota, no alarms; fsync latency normal.
- Nodes all Ready.

---

## Prevention

- Dashboards and alerts: API server request rate by verb/resource/client, latency, inflight, APF rejections; etcd DB size (alert at ~70% of quota), leader changes, fsync latency.
- Production readiness review for any component that talks to the API from every node.
- Staged rollouts for **infrastructure** components (DaemonSets, operators), not only apps.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a new observability DaemonSet issued cluster-wide LIST calls from every node. It was harmless in staging (50 nodes) but at 3,000 nodes the combined load exhausted the API servers. Root cause: load proportional to cluster size, never tested at scale; rolled to all nodes at once.

2. **Q: How did you detect it? Why not earlier?**
   A: Deploys failed and DNS errors appeared. We monitored service health, not control-plane health, so the API server's saturation wasn't alerting.

3. **Q: What did you check first?**
   A: API server request metrics by client and verb, to find who was generating the load, since reducing load is the only way back in.

4. **Q: How did you stop impact quickly?**
   A: Removed the DaemonSet using a break-glass admin path with protected priority; where that wasn't possible, blocked the agent's network access to the API server; scaled API servers.

5. **Q: How did you fix it permanently?**
   A: The agent now uses a shared informer with field selectors (only its own node's pods), it was load-tested at production node count, rolled out gradually, and has an APF FlowSchema.

6. **Q: How did you confirm?**
   A: API latency and inflight back to baseline, test deploys succeeded, no DNS errors, and the gradual re-rollout showed flat API load.

7. **Q: What if you couldn't reach the API server at all to remove the culprit?**
   A: Cut the load from outside the API: network policy/security group blocking the agent's access to the API endpoint, stopping the agent at the node level (e.g. via node config management or SSM), scaling the node group down, or asking the managed provider. That's why a break-glass plan must exist **before** the incident.

8. **Q: Why did DNS fail if pods were still running?**
   A: CoreDNS watches the API for Services/EndpointSlices. With the API down, it couldn't refresh, and as cached answers expired, lookups started failing. The data plane had a hidden dependency on the control plane.

9. **Q: What would you do differently?**
   A: Treat infra agents like risky deploys: canary them to a few nodes, watch control-plane metrics, and gate the rollout.

10. **Q: What if it happened at 10× the cluster size?**
    A: The same agent would fail faster and harder. At that size, consider splitting into multiple clusters, and require every cluster-wide component to have a per-node cost close to zero (watch caches, node-local selectors).
