# 54 · Azure Front Door, October 2025: When "Last Known Good" Isn't Good

> **Level:** ★★★ advanced
>
> **Pattern:** Valid customer configuration changes → **incompatible metadata** → a **delayed (asynchronous) crash** that passed health gates → the bad config became the **"last known good"** snapshot → global edge failure. **Families:** change (config) · shared platform · monitoring that looked healthy.
>
> **You will learn:** how Azure Front Door (AFD, Microsoft's global CDN and edge load balancer) went down globally for about **8 hours** on 29 October 2025, why its configuration **protection system didn't stop it** (the crash happened ~5 minutes **after** health checks passed), why engineers couldn't simply **roll back** (the rollback snapshot was already corrupted), how recovery worked (hand-edited snapshot, gradual reload, traffic rebalancing), and Microsoft's fixes ("**Food Taster**" validation process, synchronous processing, micro-cells).
>
> **You need:** Part 1, topics 24, 29, 38, 39.
>
> **Part 3 · Real outages · Bad deploys and config pushes** · Previous: [53 · Google Cloud Service Control 2025](53-google-cloud-service-control-2025.md) · Next: [55 · Cloudflare November 2025](55-cloudflare-nov-2025.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** AFD pushes customer configurations to all its edge sites through a **protection system**: stage by stage, advancing only on positive health signals, and saving a **"last known good" (LKG)** snapshot for rollback. A specific sequence of **valid** customer config changes, made across two different control-plane versions, produced **incompatible metadata**. Edge servers didn't crash when **loading** it; they crashed a few minutes later during **asynchronous processing** (cleanup). By then the health gates had passed, the config had spread, and the LKG had been **updated with the bad data**.

```
15:35  customer config changes (valid) → incompatible metadata
15:36  pre-production stage: looks healthy (crash is delayed, asynchronous)
15:39  reaches most edge sites, health still positive → LKG snapshot updated WITH the bad config
15:41  impact begins → 15:45 all edge sites globally
15:43  protection system auto-blocks all config changes worldwide
17:10  engineers hand-edit the LKG to remove the bad configs (can't use LKG as-is)
17:40–17:50 edited LKG deployed; edge sites slowly reload
18:30  DNS recovered; traffic manually rebalanced to healthy sites
00:05  (Oct 30) mitigated: availability and latency back to normal
```

**Picture it.** A food safety inspector tastes each dish and approves it in one minute. But the food makes people ill only after five minutes. The inspector approves it, the restaurant photographs it as "our standard recipe", and serves it everywhere. When people get sick, the "standard recipe" to go back to is the bad one.

**The one thing to know.** **Health checks only catch failures that happen before the check.** Delayed failures need longer bake times or **synchronous** processing — and your rollback point must **not** be updated until you're sure.

**You've got it if you can:** explain why the protection system passed the bad config, and why rollback wasn't possible.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

Between 15:41 UTC on 29 October and 00:05 UTC on 30 October 2025, customers and Microsoft services using Azure Front Door and Azure CDN experienced connection timeouts and DNS resolution errors. A sequence of valid customer configuration changes made across two control-plane build versions produced incompatible metadata that exposed a latent data-plane bug, crashing edge servers during asynchronous processing — after the staged rollout's health checks had already passed. Because the "last known good" snapshot had also been updated with the bad metadata, Microsoft manually edited it, redeployed it, and gradually recovered the edge fleet.

## Impact

| Item | Detail (from Microsoft's PIR, tracking ID YKYN-BWZ) |
|---|---|
| Window | **15:41 UTC 29 Oct → 00:05 UTC 30 Oct 2025** (about 8 h 24 min); partial recovery from 18:30 |
| Symptoms | Connection timeouts, DNS resolution issues, latency during recovery |
| Azure services (examples listed) | Azure Portal, App Service, Azure SQL Database, Azure Databricks, Azure Communication Services, Azure Maps, Azure Marketplace, AD B2C and others |
| Microsoft services | Microsoft 365, Dynamics 365 / Power Platform, Microsoft Entra ID components, Defender, Sentinel, Purview, Copilot for Security and others; customers' ability to open support cases |
| Customer sites | Any customer site using AFD/Azure CDN; actual impact "varied between customers and resources" |
| Cost | **Not publicly disclosed** |

## Timeline (UTC)

| Time | Event |
|---|---|
| 15:35 | Problematic customer configuration change made (corrupt metadata introduced) |
| 15:36 | Applied in pre-production stage; monitoring still healthy |
| By 15:39 | Config at most edge sites with positive health signals; **LKG updated with it** |
| ~15:41 | Asynchronous crash appears (about 5 minutes after the change) — after the config had already passed the protection safeguards |
| 15:41 | Customer impact begins (pre-production first) |
| 15:43 | Protection system automatically blocks all new and in-flight config changes worldwide |
| 15:45 | Data-plane impact on all edge sites globally |
| 15:48 | Detected by monitoring; investigation starts |
| 16:15 | Focus moves to AFD configuration changes |
| 16:18 | First public status page post; 16:20 targeted Service Health notifications |
| 17:10 | Start manually editing the latest LKG to remove problematic configs |
| 17:26 | **Azure Portal fails away from AFD** |
| 17:30 | All customer config propagation to the data plane blocked |
| 17:40 → 17:50 | Edited LKG deployed; available at all edge sites, which reload gradually |
| 18:30 | AFD DNS servers recovered; traffic manually rebalanced to healthy sites |
| 20:20 | Enough sites recovered; switch back to automatic traffic management |
| 00:05 (30 Oct) | Mitigation confirmed: availability and latency at pre-incident levels |
| 2–5 Nov | Customer config queue drained; temporary block on AFD config changes lifted 5 Nov |

## How it happened

```
 Control plane (two build versions in play)
   customer config A (version X) + customer config B (version Y)  → each valid alone
   → combined: INCOMPATIBLE metadata
                     │
 Protection system: stage 1 → (≈1 min wait, health OK) → stage 2 → ... → global (5–10 min)
                     │                                               │
                     │   health checks pass because crash is DELAYED │  LKG snapshot ← updated with bad config
                     ▼                                               ▼
 Edge server: load config ✓ → convert → ... asynchronous cleanup step → latent bug (reference counting)
                     → crash of the data-plane master process → whole node stops serving
                     → AFD's internal DNS also runs on edge sites → DNS errors too
```

From Microsoft's PIR and its December 2025 lessons-learned blog:
- The changes "themselves were valid and non-malicious".
- The bug "escaped detection due to a gap in our pre-production validation, since not all features are validated across different control plane build versions."
- The crash happened during **asynchronous** processing, so the config "passed through the protection safeguards".
- When the **master** process on a node crashes, it takes down the whole data plane on that node.

## Why it spread

- **Global consistency by design:** AFD deploys configurations to all edge sites.
- **Health gates too early:** each stage waited about a minute; the crash took about five.
- **Rollback point poisoned:** "Since the latest 'last known good' (LKG) version was updated with the conflicting metadata, we chose not to revert to it."
- **Slow node recovery:** recovering a crashed node meant reloading hundreds of thousands of configurations (the blog says about 4.5 hours in the October incidents).
- **DNS on the same edge:** AFD's DNS service ran on AFD edge sites, so DNS failed too.
- **Microsoft's own services depended on AFD**, including parts of the Azure Portal (some Portal features like Marketplace had no fallback).

## How they found it

- The **protection system** detected widespread data-plane problems and blocked changes automatically at 15:43.
- **Monitoring alerts** at 15:48 started the investigation; focus moved to config changes at 16:15.
- Early health signals in pre-production looked **healthy** because the crash was delayed.

## What they did to fix it

1. Automatic global **block** on config changes (15:43), then an explicit block on all customer config propagation (17:30) so recovery wouldn't re-introduce the bad config.
2. **Hand-edited the LKG** to remove the problematic configurations (didn't roll back to it, or to older LKGs).
3. Deployed the edited LKG to all edge sites (17:40–17:50); sites reloaded configs **gradually**.
4. **Rebalanced traffic manually** to healthy sites to avoid overload, then back to automatic at 20:20.
5. **Failed the Azure Portal away from AFD** (17:26); Entra, Intune portals and AD B2C also failed over.

## Permanent fix (from the PIR and the December 2025 blog)

| Repair item | Status as published |
|---|---|
| Fixed the control-plane incompatibility and the data-plane bug | Completed |
| **Synchronous** processing of each config before advancing to production stages; incompatible metadata detectable within seconds at every stage | Completed |
| More rollout stages with **extended bake time** | Completed |
| Protection system "always-on" — no bypass, including internal cleanup | Completed (blog) |
| One-click revert to **any** previous LKG version | Completed (blog) |
| Master crash no longer takes down workers (data-plane decoupling) | Completed (blog) |
| "**Food Taster**": an isolated process on each server validates config before the real processes load it | Expected January 2026 (blog) |
| Restore customer configs within one hour via local caching; target ~10 minutes | Completed / estimated March 2026 |
| Cross-version compatibility testing and fuzzing | Estimated February 2026 |
| Critical Microsoft services (Portal, Communication Services, Marketplace, support ticket creation) moved to **active-active with fail-away** | Completed |
| **Micro-cell** segmentation of the AFD data plane | Estimated June 2026 |

(Statuses are as Microsoft published them; check the linked pages for later updates.)

**Related incident:** on **9 October 2025** (tracking ID QNBQ-5W8), a different defect caused a similar failure mode; that time a **manual cleanup bypassed** the protection system. Microsoft says the two were not directly related.

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Canary/health gates that pass in 60 seconds while failures appear later (memory leak, GC, async jobs) | Bake times long enough for delayed failures; watch error **and** crash/restart rates during bake ([SD 34](../../System-Design/notes/34-deployment-strategies.md)) |
| "Last known good" marked automatically right after deploy | Only promote a version to "known good" after a full bake period; keep several older versions |
| Shared ingress / API gateway config from many teams | Validate configs in an isolated process before reload (`nginx -t`-style); per-tenant isolation |
| DNS for your services served from the same platform that might fail | Separate failure domains for DNS |
| Your admin portal depends on the CDN | Fail-away path for management access |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Incompatible metadata from valid changes made across two control-plane versions, which triggered a latent data-plane bug that crashed edge servers asynchronously. Contributing: validation gaps across control-plane versions, health gates shorter than the time to failure, and an LKG snapshot updated before the config was proven safe.

2. **Q: How was it detected, and why not earlier?**
   A: The protection system and monitoring caught it within minutes of impact. It wasn't caught earlier because the crash happened after health checks had passed.

3. **Q: What did they check first?**
   A: After initial alerts, the investigation moved to AFD configuration changes by 16:15.

4. **Q: How did they stop the impact?**
   A: Block all config changes, hand-edit the LKG to remove the bad configs, deploy it, reload nodes gradually, rebalance traffic.

5. **Q: Why couldn't they just roll back?**
   A: The LKG had already been updated with the bad metadata, so "rollback" would re-apply the problem; older LKGs were also not used.

6. **Q: How did they confirm recovery?**
   A: Availability and latency back to pre-incident levels at 00:05 UTC, then a controlled drain of the config queue before unblocking changes.

7. **Q: What if the edited LKG had also crashed nodes?**
   A: Recovery would restart from an even smaller config set (or older versions), with traffic shifted to whatever capacity was healthy. This is why the fix includes one-click revert to any previous LKG and an isolated validator.

8. **Q: How would you prevent this class of incident?**
   A: Synchronous validation in an isolated process before loading, bake times longer than the slowest failure mode, cross-version compatibility tests, and cells to limit blast radius.

9. **Q: What would you do differently?**
   A: Never update the rollback point automatically during a rollout; promote it only after the whole fleet has been healthy for a defined bake time.

10. **Q: What if this happened to the CDN in front of your company?**
    A: Your sites and maybe your login and admin portals go down. Keep a fail-away path (second CDN or direct origin, with DNS ready to switch), as Microsoft did for the Azure Portal.

## Source

- Microsoft Azure status history, "Post Incident Review (PIR) – Azure Front Door – Connectivity issues across multiple regions", Tracking ID YKYN-BWZ: <https://azure.status.microsoft/en-us/status/history/?trackingId=YKYN-BWZ>
- Microsoft Azure Networking Blog, "Azure Front Door: Implementing lessons learned following October outages" (18 December 2025): <https://techcommunity.microsoft.com/blog/azurenetworkingblog/-/4479416>
