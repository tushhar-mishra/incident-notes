# 28 · Network and Firewall Changes

> **Level:** ★★ intermediate
>
> **You will learn:** how network changes cause outages: **security group / firewall rules**, Kubernetes **NetworkPolicy**, **route tables** and removed routes, **NAT gateway** problems, **MTU** mismatches, and **BGP** (Border Gateway Protocol) explained in one page; the error **fingerprints** (timeout vs refused vs reset vs "no route"); the commands to test each hop (`ss`, `nc`, `curl`, `traceroute`, `tcpdump`); and how to change networks safely.
>
> **You need:** topics 01–27. Helpful: [Linux 058 · How networks work](../../linux-mastery-rhel/notes/058-how-networks-work.md), [Linux 061 · Routing](../../linux-mastery-rhel/notes/061-routing.md), [Linux 071 · firewalld](../../linux-mastery-rhel/notes/071-firewalld.md), [Linux 096 · Packet capture](../../linux-mastery-rhel/notes/096-packet-capture.md), [Docker 30 · Network troubleshooting](../../Docker/notes/30-network-troubleshooting.md), [K8s 61 · Network policies](../../CKA-Certification-Course-2025/notes/61-network-policies.md), [SD 08 · Cloud network building blocks](../../System-Design/notes/08-cloud-network-building-blocks.md).
>
> **Part 2 · Recurring failures · D. Network, DNS, certificates** · Previous: [27 · Expired TLS certificates](27-expired-tls-certificates.md) · Next: [29 · Load balancer and health checks](29-load-balancer-and-health-checks.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Packets need a **route** to the destination and must be **allowed** by every firewall on the way (host firewall, security group, NACL, NetworkPolicy, cloud firewall). Change either, and connections fail. The **error type** tells you which layer:

| Error | Usually means |
|---|---|
| **Connection timed out** | Packets **dropped silently**: firewall/security group/NACL/NetworkPolicy, or no route back |
| **Connection refused** | Packets **arrived**, but nothing listens on that port (app down, wrong port, wrong bind address) |
| **Connection reset** | Something actively killed the connection: app crashed mid-request, LB/proxy timeout, firewall reset |
| **No route to host** | No route, or an ICMP "host unreachable" (host down, ARP failure, REJECT rule) |
| **Hangs after connecting / large requests hang** | **MTU** problem (big packets dropped; small ones fine) |

**Picture it.** Posting a letter. Timeout = the letter vanished (someone threw it away). Refused = it arrived but nobody lives there. Reset = the resident tore it up. No route = the post office says "no such street".

**The one thing to know.** Run **`nc -zv host port`** (or `curl -v`) from the **source** of the failing traffic, and read the error type. That alone halves the search.

**Try it now:** `nc -zv google.com 443` (open), `nc -zv -w 3 10.255.255.1 443` (timeout), `nc -zv localhost 1` (refused).

**You've got it if you can:** debug "service A can't reach service B" hop by hop and name the layer.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| App logs | `connect: connection timed out`, `i/o timeout`, `ECONNREFUSED`, `ECONNRESET`, `EHOSTUNREACH` |
| Pattern | Starts right after a network/IaC change; affects one **path** (A→B), not everything |
| Kubernetes | Pods can't reach DB after a NetworkPolicy was applied; DNS fails if egress to CoreDNS is blocked |
| Cloud | Instances can't reach the internet (NAT gateway/route), cross-VPC traffic fails (peering/transit routes) |
| Partial | Only large responses fail (MTU); only new connections fail (conntrack table full) |
| Internet-scale | Whole networks disappear from the internet (BGP withdrawals, topic 59) |

---

## Analogy

**Posting a letter** (Core).

---

## How it happens

### Common change-caused failures

| Change | Effect |
|---|---|
| Security group rule removed/tightened (often by Terraform "cleanup") | Timeouts on that port |
| **Default-deny NetworkPolicy** applied without allowing DNS (UDP/TCP 53 to kube-dns) and app traffic | Everything in the namespace breaks, including name resolution |
| Route table change (wrong target, deleted route) | Timeouts or no route; asymmetric routing (replies take a different path and get dropped) |
| NAT gateway deleted/out of ports | Private instances can't reach the internet (package installs, APIs, registries) |
| VPN/peering/transit gateway change | Cross-network traffic fails |
| Host firewall reload (firewalld/iptables) | Docker/Kubernetes rules flushed (Docker 52 scenario) |
| OS update restarting network services | Routing rules added by the CNI removed (topics 63, 64) |
| MTU change (tunnels, VPN, jumbo frames) | Small packets OK, big packets dropped; TLS handshakes hang |
| BGP announcement/withdrawal change | Prefixes disappear from (or are wrongly announced to) the internet |

### BGP in one page

The internet is made of **autonomous systems** (ASes): networks run by one organisation (an ISP, Facebook, Cloudflare), each with an AS number. **BGP** is how ASes tell each other **"I can reach these IP prefixes, send traffic for them to me"** (an **announcement**) or **"I can't any more"** (a **withdrawal**).

```
Cloudflare AS13335: "104.16.0.0/13 is reachable via me"  ──►  ISPs update routing tables  ──► users' traffic flows
Withdraw that announcement  ──►  ISPs remove the route  ──► nobody on the internet can reach those IPs
```

Failures: withdrawing your own prefixes by mistake (Facebook 2021, topic 59; Cloudflare BYOIP 2026, topic 74), announcing prefixes you shouldn't (**route leak**: Cloudflare, January 2026, topic 60), or a bad routing policy pushed to many routers at once. Inside data centres and Kubernetes, CNIs such as Calico can also use BGP to share pod routes.

---

## How to find it

**First:** from the source, test the exact destination and port, and read the error.

```bash
nc -zv -w 3 db.internal 5432
```
```
nc: connect to db.internal (10.0.12.40) port 5432 (tcp) timed out: Operation now in progress
```
- **Timed out** → likely a firewall/SG/NACL/NetworkPolicy dropping, or routing. (Refused would mean the DB isn't listening.)

```bash
curl -sv --connect-timeout 3 http://orders:8080/healthz
ip route get 10.0.12.40          # which interface/gateway the kernel will use
traceroute -T -p 5432 10.0.12.40 # TCP traceroute: where do packets stop?
```

**Second:** check the rules on every layer between source and destination.

```bash
# AWS security groups and NACLs of the destination
aws ec2 describe-security-groups --group-ids sg-0abc --query 'SecurityGroups[].IpPermissions'
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-0def
# VPC Reachability Analyzer can test a path end to end
# Recent changes
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RevokeSecurityGroupIngress --max-results 10

# Kubernetes
kubectl get networkpolicy -n shop
kubectl describe networkpolicy default-deny -n shop

# Host
sudo iptables -S | head -50     # or: sudo nft list ruleset | head -50
sudo firewall-cmd --list-all
```

**Third:** watch the packets on the destination.

```bash
sudo tcpdump -ni any port 5432 and host 10.0.3.15
```
- **Nothing arrives** → blocked or misrouted before the host.
- **SYN arrives, no SYN-ACK** → host firewall or app not listening.
- **SYN-ACK sent but source never gets it** → return path (asymmetric routing, NACL outbound/ephemeral ports).

MTU check:
```bash
ping -M do -s 1472 10.0.12.40     # 1472 + 28 header bytes = 1500; lower until it works
```

---

## How to fix it

### Immediate

- **Revert the network change** (Terraform revert + apply, restore the SG rule, delete the new NetworkPolicy, restore the route).
- For NetworkPolicy, quickly add an allow rule (DNS + required paths) if reverting is slower.
- Restart the component that rebuilds rules (kube-proxy, CNI agent, Docker) if a reload flushed them.
- BGP: re-announce the prefixes (requires access to the routers — plan out-of-band access).

### Permanent

- Network changes as **code** with `plan` review; policy checks for dangerous diffs (e.g. "removing an ingress rule on a production DB SG") ([Terraform 54 · Policy as code](../../Terraform/notes/54-policy-as-code.md)).
- **Staged** network changes (one AZ/router/cluster first).
- NetworkPolicy templates that **always** allow DNS; test policies in staging with real traffic.
- Out-of-band management network for routers/servers.
- Document required flows (who talks to whom on which port).

---

## How to verify the fix

- `nc`/`curl` from the real source succeeds; tcpdump shows complete handshakes.
- Error rate normal; connection counts back to baseline.
- `terraform plan` shows no drift (the fix is in code, not just in the console).

---

## Prevention

- Synthetic checks for critical internal paths (app → DB, app → cache, pod → DNS, node → registry).
- Alerts on connection errors per destination (`ECONNREFUSED`/timeouts by dependency).
- Change review rules for network IaC; peer review of SG/NACL/policy diffs.
- BGP: RPKI (Resource Public Key Infrastructure) and prefix filters to prevent leaks; staged router config rollouts.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a Terraform cleanup removed a "redundant" security group rule that was the only rule allowing the app subnet to reach the DB. Root cause: no record of required flows, and the plan diff wasn't checked against them.

2. **Q: How did you detect it? Why not earlier?**
   A: App errors (timeouts to DB). Not earlier because the change looked harmless in review and there was no synthetic app→DB check.

3. **Q: What did you check first?**
   A: `nc -zv` from an app pod to the DB, because the error type (timeout vs refused) tells me whether to look at the network or the DB process.

4. **Q: How did you stop impact quickly?**
   A: Re-added the rule (via Terraform revert and apply), confirmed connectivity.

5. **Q: Permanent fix?**
   A: Documented flows as tests (a policy check fails the plan if a required flow is removed) and a synthetic connectivity check.

6. **Q: How did you confirm?**
   A: Connectivity from all app subnets, errors gone, and the plan shows no drift.

7. **Q: What if reverting didn't help?**
   A: Then the change wasn't the cause, or another layer also changed. Test hop by hop with tcpdump on both sides to find where packets stop, and check routes and NACLs (which are stateless and need return-traffic rules).

8. **Q: Timeout vs connection refused — why does it matter?**
   A: Refused proves the packet reached the host and nothing is listening (app/port problem). Timeout means packets are dropped or lost (firewall/routing). It decides which team and which layer to look at.

9. **Q: What would you do differently?**
   A: Test network changes with synthetic flows before and after the apply, in a staged way.

10. **Q: What if you applied a default-deny NetworkPolicy and everything broke including DNS?**
    A: Default-deny blocks egress to CoreDNS too. Immediately add an egress rule allowing UDP/TCP 53 to kube-dns, then add the specific app flows. Long-term: a policy template that always includes DNS, tested in staging.
