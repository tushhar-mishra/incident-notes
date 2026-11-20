# 65 · OpenAI, December 2024: A Telemetry Service Locks Everyone Out

> **Level:** ★★★ advanced
>
> **Pattern:** A **new observability service** (meant to improve reliability) makes **every node** run **expensive Kubernetes API calls** whose cost grows with cluster size → **API servers overwhelmed** in the largest clusters → **DNS-based service discovery** fails as caches expire → engineers **can't reach the control plane** to remove the cause. **Families:** change · control-plane overload (topic 22) · DNS dependency (topic 26) · monitoring causing the outage (topic 44).
>
> **You will learn:** how a telemetry service tested in staging took down the control plane of most of OpenAI's **large** Kubernetes clusters, why **DNS caching hid the problem** for ~20 minutes (letting the rollout continue), the "**locked out**" problem and the three parallel ways they regained control, and OpenAI's fixes: **phased rollouts that watch cluster health**, **break-glass** API access, and **decoupling** the data plane from the control plane.
>
> **You need:** Part 1, topics 22, 26, 44.
>
> **Part 3 · Real outages · Platform, OS and Kubernetes** · Previous: [64 · Heroku 2025](64-heroku-2025.md) · Next: [66 · AWS Kinesis 2020](66-aws-kinesis-2020.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** OpenAI runs hundreds of Kubernetes clusters. To improve observability, they deployed a new telemetry service to collect control-plane metrics. Its configuration made **every node** execute resource-intensive Kubernetes API operations whose cost **scaled with cluster size**. In the biggest clusters, thousands of nodes did this at once, and the **API servers collapsed**. Running pods kept running, but **DNS** (which needs the control plane) started failing as cached records expired. To remove the telemetry service, engineers needed the API server — which was overloaded.

```
14:23 PST merge → 14:51–15:20 applied to all clusters
   every node → expensive K8s API operations (cost ∝ cluster size)
   → API servers overwhelmed in large clusters → control plane down
   → CoreDNS can't refresh; cached records keep things working ~20 min ... then expire
   → services can't find each other → API, ChatGPT, Sora degraded/unavailable from 15:16
   → fix = delete the service ... but kubectl needs the overloaded API server ("locked out")
   → scale down clusters + block admin API access + scale up API servers (in parallel)
   → 19:38 PST all clusters recovered
```

**Picture it.** A city installs a new traffic counter at every junction. Each counter phones the central control room every few seconds and asks for a full list of every car in the city. With a small town it's fine. In a megacity the control room's phones jam — and the control room is also where you'd go to switch the counters off.

**The one thing to know.** **Agents that run on every node multiply their cost by node count.** Test them at **production scale**, roll them out **slowly while watching the control plane**, and keep a **break-glass** path into the API that doesn't depend on the API being healthy.

**You've got it if you can:** explain why staging and DNS caching both hid the problem, and the three mitigation workstreams.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 11 December 2024, OpenAI deployed a new telemetry service whose configuration caused every node in each cluster to run resource-intensive Kubernetes API operations that scaled with cluster size. In the largest clusters, the API servers were overwhelmed and the control plane went down; DNS-based service discovery failed as cached records expired, causing significant degradation or unavailability of all OpenAI services from 3:16 PM to 7:38 PM PST. Engineers regained control by reducing API load and scaling API servers in parallel, then removed the service.

## Impact

| Item | Detail (from OpenAI's write-up) |
|---|---|
| Window | "Between 3:16 PM PST and 7:38 PM PST on December 11, 2024, all OpenAI services experienced significant degradation or complete unavailability" |
| API | Substantial recovery 5:36 PM; full recovery across all models 7:38 PM |
| ChatGPT | Substantial recovery 5:45 PM; full recovery 7:01 PM |
| Sora | Fully recovered 7:01 PM |
| Cause type | Not a security incident or a recent launch |
| Clusters affected | "most of our large clusters" (exact count **not publicly disclosed**) |

## Timeline (PST)

| Time | Event |
|---|---|
| Dec 10 | New telemetry service deployed to a staging cluster; works as expected |
| Dec 11, 2:23 PM | Change merged; deployment pipeline triggered |
| 2:51–3:20 PM | Change applied to all clusters |
| 3:13 PM | Alerts fire, notifying engineers |
| 3:16 PM | Minor customer impact begins; root cause identified |
| 3:27 PM | Engineers begin moving traffic away from impacted clusters |
| 3:40 PM | Maximum customer impact |
| 4:36 PM | First cluster recovered |
| 5:36 / 5:45 PM | Substantial recovery for API / ChatGPT |
| 7:01 PM | ChatGPT and Sora fully recovered |
| 7:38 PM | All clusters recovered |

## How it happened

```
              ┌──────────────── Kubernetes cluster (thousands of nodes) ───────────────┐
              │  node 1: telemetry agent ──► expensive API operation ──┐               │
              │  node 2: telemetry agent ──► expensive API operation ──┤               │
              │  ...                                                   ▼               │
              │  node N: telemetry agent ──► ...            ┌──────────────────┐       │
              │                                             │   API servers    │ OVERLOADED
              │                                             └────────┬─────────┘       │
              │  CoreDNS (needs API for Services/Endpoints) ◄────────┘ can't refresh   │
              │  pods: keep running, use CACHED DNS (~20 min) → caches expire → lookups fail
              └─────────────────────────────────────────────────────────────────────────┘
```

OpenAI: "this new service's configuration unintentionally caused every node in each cluster to execute resource-intensive Kubernetes API operations whose cost scaled with the size of the cluster." And: "DNS relies on the control plane – services don't know how to contact one another without the Kubernetes control plane."

## Why it spread

- **Cost ∝ cluster size**: harmless in staging, fatal in the largest clusters ("our testing didn't catch it").
- **Fleet-wide rollout** in about 30 minutes, monitoring **service** health but not **cluster/control-plane** health ("no precautions were taken to assess Kubernetes API server load").
- **DNS caching delayed the symptoms** by ~20 minutes, so the rollout continued "before the full scope of the problem was understood". When caches emptied, DNS load **multiplied**, adding even more control-plane load.
- **Locked out**: "In order to make that fix, we needed to access the Kubernetes control plane – which we could not do."

## How they found it

Detection worked: alerts fired at **3:13 PM**, a few minutes **before** customers were affected, and the root cause was identified at **3:16 PM**. The difficulty was not finding the cause but **being able to act on it**.

## What they did to fix it

Three workstreams **in parallel**:
1. **Scale down cluster size** → less aggregate API load.
2. **Block network access to Kubernetes admin APIs** → stop new expensive requests so API servers could recover.
3. **Scale up Kubernetes API servers** → more resources to handle pending requests and apply the fix.

Plus: shift traffic to healthy clusters where possible. Once enough control was regained, they **removed the offending service**. Some clusters stayed degraded because many services tried to download resources at once on recovery (a thundering herd, topic 37) and needed manual work.

## Permanent fix (as listed by OpenAI)

| Remediation | Pattern it addresses |
|---|---|
| **Robust phased rollouts** for all infrastructure configuration changes, with continuous monitoring of **both** service workloads and **cluster health including the control plane** | Rollouts blind to control-plane impact |
| **Fault injection testing**: data plane must survive without the control plane for longer; tests that roll out bad changes deliberately | Untested failure mode |
| **Emergency Kubernetes control plane access (break-glass)** that works when the data plane is overloading the control plane | Lockout |
| **Decouple the data plane from the control plane** so the control plane isn't load-bearing for critical workloads (e.g. DNS) | Hidden dependency |
| **Faster recovery**: caching and dynamic rate limiters for resources needed at cluster startup; regular exercises replacing a whole cluster quickly | Recovery herd |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| A DaemonSet (monitoring agent, log shipper, security agent) calling the API from every node | Use watches/informers with node-scoped field selectors; load-test at full node count; roll out to a few nodes first while watching API server metrics (topic 22, [K8s 27](../../CKA-Certification-Course-2025/notes/27-daemonsets.md)) |
| CoreDNS depending on the API | NodeLocal DNSCache, longer cache TTLs/serve-stale for resilience ([K8s 33](../../CKA-Certification-Course-2025/notes/33-cluster-dns-and-coredns.md)) |
| kubectl as the only way to fix the cluster | Break-glass: APF FlowSchema for admin users, a separate admin endpoint, cloud provider support path |
| Rollout health checks that only look at the new service | Also gate on API server latency, etcd health, DNS error rate |
| Observability changes considered "safe" | Treat monitoring changes as production changes (topic 44) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A telemetry service configuration that made every node perform expensive API operations scaling with cluster size, rolled out fleet-wide with monitoring that checked the service but not the control plane. DNS's dependency on the control plane turned control-plane overload into a full outage.

2. **Q: How was it detected, and why wasn't it caught in staging?**
   A: Alerts fired at 3:13 PM, before customer impact. Staging used a small cluster; impact only appeared above a certain size, and DNS caching delayed visible symptoms.

3. **Q: What was checked first?**
   A: The root cause was identified at 3:16 PM — the issue was acting on it, not finding it.

4. **Q: How did they stop the impact?**
   A: Three parallel workstreams to reduce API load and add API capacity, then removing the service; traffic shifted to healthy clusters meanwhile.

5. **Q: What changed permanently?**
   A: Phased rollouts with control-plane health monitoring, fault injection, break-glass access, data/control plane decoupling, faster recovery.

6. **Q: How would you confirm a similar fix?**
   A: Deploy the corrected agent to a production-size cluster gradually while watching API server request rates and latency; run a fault-injection test that takes the control plane away and check DNS and services survive.

7. **Q: What if none of the three workstreams had worked?**
   A: Rebuild clusters (OpenAI later committed to practising rapid full-cluster replacement), or move all traffic to unaffected clusters/regions.

8. **Q: How would you prevent this class of incident?**
   A: Per-node agents must have near-zero API cost per node; staged rollouts gated on control-plane health; DNS that survives control-plane loss; break-glass access.

9. **Q: What would you do differently?**
   A: Pause any fleet-wide rollout when control-plane metrics move, even if the service itself looks healthy.

10. **Q: Why is "caching delayed the impact" dangerous?**
    A: It hides a failure until after the rollout has spread everywhere, and when caches expire all at once, the recovery load spikes. Rollout bake times must exceed cache TTLs.

## Source

- OpenAI Status, "API, ChatGPT & Sora Facing Issues" — incident write-up (11 December 2024): <https://status.openai.com/incidents/ctrsv3lwd797/write-up>
