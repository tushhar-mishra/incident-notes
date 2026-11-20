# 82 · Drill: Intermittent DNS Failures in the Cluster

> **Level:** ★★ intermediate
>
> **You will practise:** the frustrating "**sometimes it fails**" DNS incident in Kubernetes: occasional `no such host` / 5-second delays, **CoreDNS** overload, **ndots** amplification, conntrack issues, and fixes like **scaling CoreDNS** and **NodeLocal DNSCache**.
>
> **You need:** topics 26, 76. Helpful: [K8s 33 · Cluster DNS and CoreDNS](../../CKA-Certification-Course-2025/notes/33-cluster-dns-and-coredns.md).
>
> **Part 4 · On-call drills** · Previous: [81 · "certificate has expired"](81-drill-certificate-expired.md) · Next: [83 · "Too many connections"](83-drill-too-many-connections.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **11:20 — PAGE:** `Elevated 5xx across several services (1–3%); logs show "dial tcp: lookup payments.shop.svc.cluster.local: i/o timeout"`

**The idea.** Several unrelated services fail **a small percentage** of requests with DNS timeouts. That points to **shared DNS** (CoreDNS) being overloaded or dropping packets.

```
many services + small % + DNS timeout → shared resolver
check CoreDNS: pods, CPU, restarts, request rate, SERVFAIL, latency
look for amplification: ndots:5 + external names → many queries per lookup; a chatty client
fix: scale CoreDNS / NodeLocal DNSCache / reduce ndots / fix the noisy client
```

**The one thing to know.** **Intermittent + many services = shared dependency.** Don't debug each service separately.

> 📖 Read only **Symptoms**, then write your first three checks.

---

## Symptoms

- 1–3% errors in several services, all with DNS timeouts or `no such host` for names that exist.
- Some requests are slow by ~5 seconds (resolver retry/timeout).
- Started around a traffic increase or after a new service went live.

## Analogy

The office receptionist (CoreDNS) answers everyone's "what's the extension for X?" questions. A new team arrives that asks five questions for every call they make. The receptionist starts missing questions, and random people across the building can't reach anyone.

## How it happens

- **CoreDNS undersized** for the cluster's query rate (too few replicas, CPU limits).
- **ndots:5 amplification**: external names (`api.stripe.com`) try 4 search domains first → many NXDOMAIN queries (topic 26).
- **A noisy client** resolving in a tight loop (no DNS caching in the app).
- **conntrack race** on UDP (parallel A/AAAA queries) → occasional 5-second timeouts.
- **Upstream resolver** slow (VPC resolver limits per ENI, e.g. AWS has a per-network-interface packet limit to the VPC DNS resolver).

### Lab setup (break it)

```bash
kubectl -n kube-system scale deploy coredns --replicas=1
kubectl -n kube-system set resources deploy coredns --limits=cpu=50m      # starve it
# A noisy client: 20 pods resolving an external name in a loop (ndots amplification)
kubectl -n shop create deployment noisy --image=busybox:1.36 --replicas=20 -- \
  sh -c 'while true; do nslookup api.github.com >/dev/null 2>&1; done'
# Watch errors from a "victim"
kubectl -n shop run victim --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do nslookup web.shop.svc.cluster.local >/dev/null 2>&1 && echo ok || echo DNSFAIL; sleep 0.5; done'
kubectl -n shop logs -f victim | uniq -c
```

## How to find it

**First — is CoreDNS healthy?**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system top pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=30
```
```
NAME                       CPU(cores)   MEMORY(bytes)
coredns-7db6d8ff4d-x8k2p   50m          28Mi        ← pinned at its 50m CPU limit
[ERROR] plugin/errors: 2 api.github.com.shop.svc.cluster.local. A: read udp ... i/o timeout
```

Prometheus: `sum(rate(coredns_dns_requests_total[5m]))`, `sum by (rcode) (rate(coredns_dns_responses_total[5m]))`, `histogram_quantile(0.99, sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))`, CPU throttling of CoreDNS pods.

**Second — who is sending the queries, and how many per lookup?**

```bash
kubectl -n kube-system edit configmap coredns     # temporarily add 'log' to the server block to see queries
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=200 | awk '{print $4}' | sort | uniq -c | sort -rn | head
kubectl -n shop exec deploy/noisy -- cat /etc/resolv.conf
```
```
search shop.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
- Many `NXDOMAIN` for `api.github.com.<search-domain>` → ndots amplification from the noisy client's pod IPs.

**Third — rule out the upstream and the network:** `dig @<upstream>` from a node; check node-level conntrack (`conntrack -S` for `insert_failed`).

## How to fix it

### Immediate

```bash
kubectl -n kube-system scale deploy coredns --replicas=4
kubectl -n kube-system edit deploy coredns     # remove the CPU limit (keep a CPU request, e.g. 100m)
kubectl -n shop scale deploy noisy --replicas=0     # if it's a misbehaving client
```

### Permanent

- **NodeLocal DNSCache** on every node (local cache, fewer upstream queries, avoids the conntrack race by using TCP upstream).
- **CoreDNS autoscaling** with cluster size (`cluster-proportional-autoscaler`) and sensible resources.
- **Lower `ndots`** for pods that mostly call external names (`dnsConfig: options: [{name: ndots, value: "2"}]`) or use FQDNs with a trailing dot.
- App-level DNS caching and connection reuse.

## How to verify the fix

- Victim loop: no `DNSFAIL` for 10+ minutes; DNS p99 latency low; SERVFAIL/timeout rate ≈ 0.
- Service error rates back to baseline; no 5-second latency outliers.

## Prevention

CoreDNS SLO (latency + error rate) with alerts; capacity scaling with cluster size; review of DNS settings in the service template.

## Follow-up questions

1. **Q: Root cause?**
   A: CoreDNS was undersized (one replica, tight CPU limit) and a new service resolved external names in a loop, amplified 4–5× by `ndots:5`. Contributing: no CoreDNS SLO alerts.
2. **Q: How did you detect it? Why not earlier?**
   A: 5xx alerts on several services. A CoreDNS latency/throttling alert would have fired first.
3. **Q: What did you check first?**
   A: Whether failures share a dependency (DNS errors across services), then CoreDNS health.
4. **Q: How did you stop impact?**
   A: Scaled CoreDNS and removed its CPU limit; scaled down the noisy client.
5. **Q: Permanent fix?**
   A: NodeLocal DNSCache, CoreDNS autoscaling, lower ndots, client-side caching.
6. **Q: How did you confirm?**
   A: Zero DNS failures in a continuous test; DNS latency and error metrics normal.
7. **Q: What if scaling CoreDNS didn't help?**
   A: Look at the upstream resolver (VPC DNS limits per ENI), node conntrack table/insert failures, network policies blocking DNS on some nodes, or a broken CoreDNS config (forwarding loop).
8. **Q: How would you prevent it?**
   A: Treat cluster DNS as a critical service with SLOs, scale with the cluster, cache on each node.
9. **Q: What would you do differently?**
   A: Add CoreDNS dashboards to the default on-call view; DNS is in the path of almost everything.
10. **Q: What if this happened at 10× traffic?**
    A: Without node-local caching, CoreDNS and the upstream resolver would saturate quickly; NodeLocal DNSCache and autoscaling are essential at scale.
