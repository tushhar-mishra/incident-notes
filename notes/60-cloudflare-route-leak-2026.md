# 60 · Cloudflare Route Leak, January 2026: A Policy That Accepted Too Much

> **Level:** ★★ intermediate
>
> **Pattern:** An **automated network policy change** removes one item from a match clause → the remaining clause is **too permissive** → the router **advertises routes it shouldn't** to transit providers and peers (a **BGP route leak**) → other networks' traffic is pulled through Miami → congestion and drops. **Families:** DNS/network · change (automation).
>
> **You will learn:** what a **route leak** is (in plain words and RFC 7908 terms), how deleting a prefix list from a JunOS policy term left `from route-type internal; then accept;`, which matched **all internal IPv6 routes**, why only **one router** and **25 minutes** limited the damage, and the protections the internet is adopting (**RFC 9234 BGP roles**, **RPKI ASPA**).
>
> **You need:** Part 1, topic 28 (BGP in one page).
>
> **Part 3 · Real outages · DNS and network** · Previous: [59 · Facebook BGP 2021](59-facebook-bgp-2021.md) · Next: [61 · Cloudflare 1.1.1.1 2025](61-cloudflare-1111-2025.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** BGP rules say: routes you learn from a **peer** or a **provider** should only be passed on to your **customers**, never to other peers or providers. Passing them on anyway is a **route leak**: you're telling the internet "send that traffic through me", when you shouldn't carry it. On 22 January 2026, Cloudflare's automation changed export policies on a Miami router to stop announcing a Bogotá data center's IPv6 prefixes. Removing the prefix list left a rule that matched **every internal route**, so the router advertised Cloudflare's internally redistributed IPv6 routes (including routes learned from other networks) to its transit providers and peers.

```
Before:  term ADV-SITELOCAL-GRE-RECEIVER  from { prefix-list 6-BOG04-SITE-LOCAL; route-type internal; }  then accept
Change:  remove prefix-list 6-BOG04-SITE-LOCAL
After:   term ADV-SITELOCAL-GRE-RECEIVER  from { route-type internal; }  then accept
         → matches ALL internal (incl. iBGP-learned) IPv6 routes → advertised to transit + peers
20:25 UTC automation runs on one Miami edge router → leak starts
20:50 operator manually reverts and pauses automation → 25 minutes
```

**Picture it.** A post office is told: "stop forwarding letters for Bogotá". The clerk deletes the word "Bogotá" from the instruction "forward Bogotá letters that come from our own staff" — leaving "forward letters that come from our own staff". Now the office forwards everything its staff touch, including other companies' post, to every courier it knows.

**The one thing to know.** **Removing a filter can widen a rule.** Automated policy changes need checks for **empty or over-broad match terms** before they're applied.

**You've got it if you can:** explain why deleting a condition made the rule match more, and why route leaks hurt other networks.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 22 January 2026, an automated routing-policy change on a single Cloudflare edge router in Miami removed a prefix list from several export policy terms, leaving terms that matched every internal route. The router advertised internally redistributed IPv6 prefixes to its BGP transit providers and peers for 25 minutes, pulling some external networks' traffic through Miami, causing backbone congestion, elevated loss and latency, and around 12 Gbps of discarded traffic at peak. An operator manually reverted the change and paused the automation.

## Impact

| Item | Detail (from Cloudflare's post) |
|---|---|
| Duration | **25 minutes** (20:25–20:50 UTC) |
| Scope | **IPv6 only** |
| Cloudflare customers | Some impact: elevated loss and higher latency from congestion on the Miami backbone (e.g. Miami–Atlanta) |
| External parties | "multiple external parties were also affected because their traffic was accidentally funnelled through our Miami data center" |
| Dropped traffic | Firewall filters discarded traffic not destined for Cloudflare/customers: "around 12Gbps" at peak |
| Number of leaked prefixes / affected networks | **Not publicly disclosed** (examples shown include a Meta prefix sent towards Lumen) |

## Timeline (UTC, 22 January 2026)

| Time | Event |
|---|---|
| 19:52 | Change that ultimately triggers the policy bug merged in the network automation repository |
| 20:25 | Automation runs on a single Miami edge router → unexpected advertisements to transit providers and peers (**impact start**) |
| 20:40 | Network team begins investigating unintended route advertisements from Miami |
| 20:44 | Incident raised |
| 20:50 | Operator manually reverts the bad change; automation paused for the router (**impact stop**) |
| 21:47 | Triggering change reverted in the code repository |
| 22:07 | Automation confirmed healthy to run again without the bug |
| 22:40 | Automation unpaused on the Miami router |

## How it happened

**What a route leak is (Cloudflare's explanation):** a route leak happens "when a network tells the broader Internet to send it traffic that it's not supposed to forward". Under "valley-free" routing, a route learned from a peer or provider should only be passed down to customers. Cloudflare classified this incident as a mix of **Type 3 and Type 4** leaks under **RFC 7908**.

```
   Lumen (provider)        Meta (peer)
        ▲                     │ Meta's prefix learned by Cloudflare
        │ ✗ should NOT        ▼
        │   be sent here   Cloudflare backbone (iBGP spreads it internally)
        │                     │
        └──── Miami router export policy: "from route-type internal; then accept"
                → advertises Meta's prefix (and others) to Lumen and other neighbours
                → some of the internet now routes that traffic via Cloudflare Miami
```

The intended change: stop announcing Bogotá (BOG04) IPv6 prefixes from Miami, because infrastructure upgrades made that forwarding unnecessary. The change removed prefix list `6-BOG04-SITE-LOCAL` from terms in nine export policies (for transit providers like Cogent, GTT, Level3/Lumen, Telia, Telefónica and for peers). On JunOS, `route-type internal` "will match any non-external route type, such as Internal BGP (IBGP) routes". With the prefix list gone, every internal IPv6 route matched. Cloudflare notes this was "very similar" to an outage it had in **2020**.

## Why it spread

- BGP announcements to transit providers propagate to the wider internet quickly, so other networks' routing changed.
- Only **one** router ran the automation, which limited the blast radius to Miami.
- Router firewall filters dropped traffic not meant for Cloudflare or its customers, which limited misuse but caused loss for those networks.

## How they found it

The team started investigating unintended route advertisements from Miami at 20:40 (15 minutes after start). **What alert or report first flagged it is not publicly disclosed.**

## What they did to fix it

1. **Manual revert** by a network operator and **paused automation** on the router (20:50).
2. Reverted the change in the code repository (21:47).
3. Verified automation healthy, then unpaused it (22:07–22:40).

## Permanent fix (as listed by Cloudflare)

| Remediation | Pattern it addresses |
|---|---|
| Patch the failure in routing-policy automation that caused the leak | The specific bug |
| **BGP community-based safeguards** that explicitly reject routes received from providers and peers on external export policies | Defence in depth independent of match terms |
| **Automatic routing policy evaluation in CI/CD** looking for empty or erroneous policy terms | Catch over-broad rules before deploy |
| Better early detection of network config issues and their effects | Detection |
| Validate vendor implementations of **RFC 9234** (BGP roles and the Only-to-Customer attribute) in preparation for rollout | Prevent local route leaks regardless of policy mistakes |
| Encourage adoption of **RPKI ASPA** (Autonomous System Provider Authorization) | Let other networks reject leaked paths |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Removing a condition from a security group rule, IAM policy, NetworkPolicy selector or firewall rule | Review the **resulting** rule, not just the diff; policy tests for "allows too much" ([Terraform 54](../../Terraform/notes/54-policy-as-code.md)) |
| Kubernetes NetworkPolicy with an empty `podSelector: {}` (matches all pods) | Linting for empty selectors; tests of intended flows ([K8s 61](../../CKA-Certification-Course-2025/notes/61-network-policies.md)) |
| Calico/MetalLB/cloud BGP configs | Change review, staged rollout per router/node, export filters |
| Network automation (Ansible for routers) | `--check --diff`, run on one device first (Cloudflare's single-router run limited the damage) ([Ansible 44 · Beyond Linux servers](../../Ansible/notes/44-beyond-linux-servers.md)) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: An automated policy change that removed a prefix list from match terms without checking what the remaining terms matched; `route-type internal` alone accepted all internal routes. Missing CI checks for over-broad terms and no independent export safeguard (communities, BGP roles) allowed it.

2. **Q: How was it detected, and why not earlier?**
   A: Investigation started 15 minutes in; the trigger signal isn't stated. A CI policy evaluation would have flagged the over-broad term before deployment.

3. **Q: What would you check first?**
   A: What the router is advertising now vs before (BGP advertised-routes per neighbour), and what changed in policies recently.

4. **Q: How did they stop the impact?**
   A: Manual revert and pausing automation on that router.

5. **Q: What changed permanently?**
   A: Automation fix, community-based reject safeguards, CI checks for empty or erroneous terms, better detection, and work towards RFC 9234 and ASPA.

6. **Q: How would you confirm the fix?**
   A: Compare advertised routes to an expected list after each change; CI tests that a policy with a removed prefix list fails validation.

7. **Q: What if the manual revert hadn't worked?**
   A: Shut down the BGP sessions to the affected neighbours from that router (or drain the router), accepting reduced capacity in Miami, to stop the leak.

8. **Q: How would you prevent this class of incident?**
   A: Multiple independent layers: correct policy, default-reject export safeguards, protocol-level roles (RFC 9234), and ecosystem checks (ASPA, filtering by providers).

9. **Q: What would you do differently?**
   A: Treat every "remove" change as potentially widening, and test the resulting policy semantically.

10. **Q: Why does a route leak hurt other companies, not just the leaker?**
    A: Other networks' traffic follows the leaked routes and passes through (or gets dropped by) the leaking network, adding latency and loss for services that had nothing to do with the change.

## Source

- Cloudflare, "Route leak incident on January 22, 2026": <https://blog.cloudflare.com/route-leak-incident-january-22-2026>
