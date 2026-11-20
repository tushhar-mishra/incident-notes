# 26 · DNS Failures

> **Level:** ★★ intermediate
>
> **You will learn:** how a DNS lookup really works (**stub resolver → recursive resolver → root → TLD → authoritative**), the error types (**NXDOMAIN**, **SERVFAIL**, **timeout**, **wrong answer**), why **TTL** and **caching** make DNS problems slow to appear and slow to go away (including **negative caching**), the classic causes (bad record change, expired domain, resolver overload, DNSSEC errors, **Kubernetes `ndots:5`**, conntrack races, CoreDNS overload), the `dig` commands that diagnose each, and why "**it's always DNS**" became a joke.
>
> **You need:** topics 01–25. Helpful: [Linux 063 · Name resolution](../../linux-mastery-rhel/notes/063-name-resolution-and-dns-client.md), [Docker 29 · Container DNS](../../Docker/notes/29-container-dns.md), [K8s 33 · Cluster DNS and CoreDNS](../../CKA-Certification-Course-2025/notes/33-cluster-dns-and-coredns.md), [SD 07 · DNS for system design](../../System-Design/notes/07-dns-for-system-design.md).
>
> **Part 2 · Recurring failures · D. Network, DNS, certificates** · Previous: [25 · Dependency and OS updates](25-dependency-and-os-updates.md) · Next: [27 · Expired TLS certificates](27-expired-tls-certificates.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** DNS (Domain Name System) turns names (`api.example.com`) into IP addresses. Almost every request starts with a DNS lookup. When DNS fails, **everything** fails, often with confusing errors that don't mention DNS at all.

```
app ──► stub resolver (/etc/resolv.conf) ──► recursive resolver (VPC resolver, CoreDNS, 1.1.1.1)
                                                 │ (cache miss)
                                                 ▼
                                   root (.) ──► TLD (.com) ──► authoritative (example.com's DNS)
```

**Four ways it fails:**

| Result | Meaning |
|---|---|
| **NXDOMAIN** | "This name does not exist" (record deleted, typo, expired domain) |
| **SERVFAIL** | "I couldn't get an answer" (authoritative unreachable, DNSSEC failure) |
| **Timeout** | No answer at all (resolver overloaded, network/firewall, packet loss) |
| **Wrong answer** | Resolves, but to the wrong/old/empty IP (bad change, stale cache) |

**Picture it.** A phone book. The phone book can be missing a page (NXDOMAIN), the librarian can be too busy to look it up (timeout), or the book can list the old number of a shop that moved (stale cache).

**The one thing to know.** DNS has **memory** (caching). A bad DNS change can take minutes or hours to **show up** everywhere and the same time to **go away** after you fix it. Know your **TTLs**.

**Try it now:** `dig +noall +answer example.com` — note the TTL number. Run it again and watch it count down.

**You've got it if you can:** tell NXDOMAIN, SERVFAIL and timeout apart with `dig`, and explain `ndots:5` in Kubernetes.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| App logs | `no such host`, `Name or service not known`, `Temporary failure in name resolution`, `getaddrinfo ENOTFOUND`, `UnknownHostException`, `i/o timeout` on `udp 10.96.0.10:53` |
| Users | Site unreachable for some users/regions (caching differences); browser `DNS_PROBE_FINISHED_NXDOMAIN` |
| Many services at once | Unrelated services failing simultaneously (shared resolver or shared dependency) |
| Kubernetes | Intermittent ~5-second delays on lookups; CoreDNS pods at high CPU; `SERVFAIL` for external names |
| Latency | Every request slow by a fixed amount (failed lookups retried after a timeout) |

---

## Analogy

The **phone book** (Core). **TTL** is the "valid until" date printed on each entry: people won't re-check the phone book until it passes.

---

## How it happens

### Common causes

| Cause | Result |
|---|---|
| **Bad record change** (wrong IP, deleted record, wrong CNAME) | Wrong answer / NXDOMAIN, spreading as caches expire |
| **Automation writes an empty or wrong record** | AWS us-east-1, October 2025 (topic 62): DynamoDB's regional endpoint record became **empty** |
| **Domain expired** or delegation (NS records) wrong at the registrar | NXDOMAIN/SERVFAIL for everything under the domain |
| **Authoritative servers unreachable** | SERVFAIL after caches expire. **Facebook 2021** (topic 59): their DNS servers withdrew their own BGP routes when they lost the backbone |
| **Resolver overload/failure** | Timeouts; a single resolver (or CoreDNS) becomes a bottleneck |
| **DNSSEC** errors (expired signatures, bad key rollover) | SERVFAIL from validating resolvers only |
| **Kubernetes `ndots:5`** | Each external lookup tries several cluster search domains first → 4–5× more queries |
| **conntrack race** (Linux, UDP) | A/AAAA queries sent in parallel from the same socket can lose a packet in a conntrack insert race → 5-second timeout and retry |
| **DNS depends on something that's down** | OpenAI 2024 (topic 65): Kubernetes DNS needed the API server |
| **Too-long TTL during a migration** | You change an IP, but clients keep the old one for hours |
| **Negative caching** | An NXDOMAIN answer is cached too (for the SOA "minimum"/negative TTL), so a record you just created can stay "missing" for a while |

### Kubernetes `ndots:5` explained

A pod's `/etc/resolv.conf` typically looks like:

```
search shop.svc.cluster.local svc.cluster.local cluster.local ec2.internal
nameserver 10.96.0.10
options ndots:5
```

`ndots:5` means: if a name has **fewer than 5 dots**, try the **search domains first**. So `api.stripe.com` (2 dots) becomes:

```
api.stripe.com.shop.svc.cluster.local   → NXDOMAIN
api.stripe.com.svc.cluster.local        → NXDOMAIN
api.stripe.com.cluster.local            → NXDOMAIN
api.stripe.com.ec2.internal             → NXDOMAIN
api.stripe.com                          → answer
```
×2 for A and AAAA = up to **10 queries** for one lookup. At scale, this overloads CoreDNS. Fixes: use a trailing dot (`api.stripe.com.`), lower `ndots` via `dnsConfig`, use **NodeLocal DNSCache**.

---

## How to find it

**First:** is it DNS? Resolve the name from the **failing** place.

```bash
# On the host / in the pod that fails
getent hosts api.internal.example.com       # uses the same path as most apps
dig api.internal.example.com
```
```
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 41234
;; AUTHORITY SECTION:
example.com.   300  IN  SOA  ns-1.example.com. hostmaster.example.com. 2026112001 7200 900 1209600 300
```
- `status: NXDOMAIN` → the name doesn't exist (according to this resolver).
- The SOA in AUTHORITY: last number (`300`) is the **negative TTL**: this NXDOMAIN is cached for up to 5 minutes.

```
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL
```
→ the resolver failed to get an answer (authoritative down, DNSSEC).

```
;; communications error to 10.96.0.10#53: timed out
;; no servers could be reached
```
→ the resolver itself isn't answering (overloaded, network, CoreDNS down).

**Second:** ask different resolvers — is it the record, or the resolver?

```bash
dig @10.96.0.10 api.internal.example.com +short      # cluster DNS
dig @169.254.169.253 api.internal.example.com +short # AWS VPC resolver (also at VPC CIDR base + 2)
dig @1.1.1.1 www.example.com +short                  # public resolver
dig @ns-1.example.com www.example.com +short          # ask the AUTHORITATIVE server directly
dig +trace www.example.com                            # walk root → TLD → authoritative
```
- Authoritative gives the right answer, resolver gives the wrong one → **stale cache**; wait for TTL or flush.
- Authoritative gives the wrong answer → **the record itself is wrong**; fix the zone.

**Third (Kubernetes):** is CoreDNS healthy?

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system top pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50 | grep -iE "error|timeout|SERVFAIL"
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default
```
Metrics: `coredns_dns_requests_total`, `coredns_dns_responses_total{rcode="SERVFAIL"}`, `coredns_forward_request_duration_seconds`.

---

## How to fix it

### Immediate

- **Revert the record change** (Route 53 change history, Terraform state, DNS provider audit log). Remember clients cache the bad answer for the old TTL.
- **Overloaded resolver:** scale CoreDNS (`kubectl -n kube-system scale deploy coredns --replicas=6`), or enable NodeLocal DNSCache; find the client generating the query flood.
- **Authoritative down:** fail over to a secondary DNS provider if you have one.
- **Stale caches:** flush what you control (`resolvectl flush-caches`, restart CoreDNS for its cache, restart apps with JVM DNS caches); for the rest, wait out the TTL.
- **Expired domain:** renew at the registrar immediately (and expect propagation delays).

### Permanent

- DNS changes go through **code review** (Terraform/OctoDNS/external-dns) with validation and staged rollouts.
- **Lower TTLs before planned migrations** (days ahead), raise them after.
- Domain **auto-renew** + registrar lock + expiry monitoring.
- Kubernetes: NodeLocal DNSCache, CoreDNS autoscaling (`cluster-proportional-autoscaler`), tuned `ndots`, trailing dots for external FQDNs in config.
- Applications: cache DNS sensibly, retry on failure, and **don't** cache forever (old JVMs).
- Consider serving stale DNS answers when upstream fails (CoreDNS `cache` plugin `serve_stale`) — this is what helped Cloudflare's 1.1.1.1 users during the .de DNSSEC problem in May 2026.

---

## How to verify the fix

- `dig` from several places (pod, node, public resolvers, authoritative) returns the correct answer with the expected TTL.
- CoreDNS error rate and latency normal; no `SERVFAIL` spikes.
- Application errors gone; affected regions/users recovered (some may lag by the old TTL).

---

## Prevention

- **External synthetic checks** resolving your key names from several regions/resolvers.
- Alerts: CoreDNS SERVFAIL rate, request latency, pod CPU; domain and certificate expiry.
- Two DNS providers for critical public zones (with care: both must be updated consistently).
- Know your **hidden DNS dependencies** (what does DNS itself depend on?).

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a Terraform change to a Route 53 record set the wrong target, and the change wasn't tested; with a 1-hour TTL the damage spread slowly and recovery after revert was slow. Root cause: unreviewed DNS change plus long TTL; contributing: no synthetic DNS check.

2. **Q: How did you detect it? Why not earlier?**
   A: Users in some regions reported errors as caches expired. A synthetic check resolving the name from outside would have caught it immediately.

3. **Q: What did you check first?**
   A: `dig` from the failing location, then the authoritative server directly, to separate "record is wrong" from "resolver/cache is wrong".

4. **Q: How did you stop impact quickly?**
   A: Reverted the record and flushed the caches we control; for clients we don't control we had to wait out the TTL, so we communicated that.

5. **Q: Permanent fix?**
   A: DNS changes through reviewed IaC with a plan diff, a lower TTL on critical records, and synthetic resolution checks.

6. **Q: How did you confirm?**
   A: Correct answers from multiple public resolvers and regions; error rates normal; synthetic check green.

7. **Q: What if reverting didn't help?**
   A: Then either caches still hold the bad answer (wait/flush), or the problem is not the record: check the resolver path, delegation (`dig +trace`), DNSSEC, and network access to port 53 (UDP and TCP).

8. **Q: Intermittent 5-second DNS delays in Kubernetes — what's going on?**
   A: Classic sign of the conntrack race on parallel UDP A/AAAA queries, or CoreDNS overload. Fixes: NodeLocal DNSCache (uses TCP upstream and avoids the race), `single-request-reopen` option in older setups, scaling CoreDNS, and reducing `ndots` query amplification.

9. **Q: What would you do differently?**
   A: Lower TTLs before risky changes, and monitor DNS from outside like a user.

10. **Q: What if this happened at 10× traffic in Kubernetes?**
    A: CoreDNS would be the first to saturate, with `ndots` multiplying queries. Autoscale CoreDNS with the cluster size, enable NodeLocal DNSCache on every node, and watch CoreDNS latency as a key SLI.
