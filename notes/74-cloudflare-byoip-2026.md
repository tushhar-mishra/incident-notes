# 74 · Cloudflare BYOIP, February 2026: An Empty Flag Meant "All"

> **Level:** ★★ intermediate
>
> **Pattern:** A new **cleanup automation** calls an API with a flag that has **no value** → the server treats it as "**all prefixes**" → the task deletes customers' **IP prefixes and their dependent objects** → ~1,100 BYOIP prefixes withdrawn from the internet. **Families:** automation gone wrong · DNS/network (BGP withdrawals) · API design.
>
> **You will learn:** what **BYOIP** (Bring Your Own IP) is, how `?pending_delete` with no value became "return everything", why staging and tests didn't catch it, why restoration had **three different paths** (some customers could fix it themselves, others needed a **global configuration push**), and Cloudflare's fixes (API schema validation, operational snapshots, a **circuit breaker** for fast or broad withdrawals).
>
> **You need:** Part 1, topics 24, 28, 40.
>
> **Part 3 · Real outages · Automation and security** · Previous: [73 · GitHub 2026 mini-cases](73-github-2026-mini-cases.md) · Next: [75 · CircleCI 2023](75-circleci-2023.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **BYOIP** lets customers bring their own IP address ranges to Cloudflare, which then **announces** them to the internet via BGP. Cloudflare automated a regular manual request: removing BYOIP prefixes a customer had asked to delete. The new sub-task asked the Addressing API for prefixes with `?pending_delete` — **without a value**. The server read the empty value and returned **all BYOIP prefixes**, not just those pending deletion. The task then began **deleting** them, with their **service bindings**, until an engineer stopped it.

```
client:  GET /v1/prefixes?pending_delete          (flag present, value empty)
server:  v := query.Get("pending_delete")  → ""   → treated as a request for ALL BYOIP prefixes
task:    queue all returned prefixes for deletion → delete prefixes + dependent objects (service bindings)
17:56 UTC withdrawals start → 18:46 engineer kills the task → ~1,100 of 4,306 BYOIP prefixes withdrawn
restoration: dashboard toggle (most) / partial / global config push to re-apply bindings (slowest) → 23:03
```

**Picture it.** A storage company's cleaner is told "throw away boxes marked for disposal". The instruction form has a "marked for disposal: ____" field left blank, and the warehouse system reads a blank as "everything". The cleaner starts emptying the whole warehouse until someone notices.

**The one thing to know.** **Empty must never mean "all"** in a destructive path. Validate inputs strictly, and put **rate and scope limits** (circuit breakers) on any automation that deletes or withdraws.

**You've got it if you can:** explain why tests missed it and why some customers recovered in minutes while others waited hours.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 20 February 2026, a newly released sub-task in Cloudflare's Addressing API, meant to automate the removal of BYOIP prefixes customers had requested to delete, called the API with a `pending_delete` flag that had no value; the server interpreted this as a request for all BYOIP prefixes. The task began deleting BYOIP prefixes and their dependent objects, withdrawing about 1,100 of 4,306 BYOIP prefixes from the internet between 17:56 and 18:46 UTC. Customers lost connectivity to services using those ranges until prefixes and service bindings were restored, the last at 23:03 UTC.

## Impact

| Item | Detail (from Cloudflare's postmortem) |
|---|---|
| Prefixes | About **1,100** prefixes withdrawn (≈ **25% of 4,306** BYOIP prefixes) between 17:56 and 18:46 UTC |
| Products | Core CDN and security services, **Spectrum**, **Dedicated Egress**, **Magic Transit** — any product using BYOIP to advertise to the internet |
| Symptoms | "BGP Path Hunting", then connection timeouts or failures; the 1.1.1.1 *website* returned HTTP 403 "Edge IP Restricted" (1.1.1.1 DNS resolution, including DoH, was **not** affected) |
| Duration | Cloudflare states 6 h 7 min; impact ended 23:03 UTC (the post doesn't reconcile the stated total with its timestamps) |
| Not all customers | Changes applied iteratively, so not all BYOIP customers were affected |
| Customers / cost | **Not publicly disclosed** |

## Timeline (UTC)

| Time | Event |
|---|---|
| 5 Feb 21:53 | Broken sub-process merged |
| 20 Feb 17:46 | Addressing API release with the broken sub-process completes |
| 17:56 | Sub-process starts executing; prefixes start being withdrawn (**impact start**) |
| 18:13 | Cloudflare engaged for failures on one.one.one.one |
| 18:18 | Internal incident declared |
| 18:21 | Addressing API team paged |
| 18:46 | Engineer terminates the sub-process and disables its regular execution |
| 19:11 | Mitigation begins |
| 19:19 | Customers told they can re-advertise prefixes from the dashboard |
| 19:44 | Database recovery methods started for removed prefixes |
| ~20:20–20:30 | About 800 prefixes restored; final mitigation process begins |
| 21:08 | Global machine configuration rollout begins |
| 23:03 | Configuration update complete; remaining ~300 prefixes restored; **impact ends** |

## How it happened

```
Addressing API (source of truth for customer IPs) → changes propagate immediately to the edge
   customers bind products to BYOIP ranges via "service bindings"

New cleanup sub-task (runs regularly):
   GET /v1/prefixes?pending_delete       ← no value
   server: if v := Query().Get("pending_delete"); v != "" { ...return only pending-delete prefixes... }
           else → (as Cloudflare explains) interpreted as a request for ALL BYOIP prefixes
   task: delete each returned prefix + dependent objects (service bindings)
   → edge stops announcing those prefixes → internet traffic for them no longer reaches Cloudflare
```

Why tests didn't catch it (Cloudflare's words): staging "contains data that matches Production as closely as possible, but was not sufficient"; test coverage "for this scenario ... was incomplete"; testing "did not cover a scenario where the task-runner service would independently execute changes to user data without explicit input."

## Why it spread

- The task ran **iteratively** through prefixes until stopped (50 minutes).
- **Immediate propagation** of Addressing API changes to the edge.
- **Dependent objects** (service bindings) were deleted too, which made restoration harder than just re-announcing.
- Preventative work (better configuration change support) was already in progress but **not fully deployed**.

## How they found it

Cloudflare became engaged at 18:13 for failures on `one.one.one.one`; an incident was declared at 18:18 and the Addressing API team paged at 18:21. What first alerted them beyond this is **not publicly disclosed**.

## What they did to fix it

1. **Killed the sub-process** and disabled its regular execution (18:46).
2. Restoration by impact type:
   - **Prefixes withdrawn only** (most customers): customers could re-advertise via the dashboard (guidance at 19:19).
   - **Prefixes withdrawn + some bindings removed:** some prefixes could be toggled, others not.
   - **Prefixes withdrawn + all bindings removed:** no self-service; needed a **global configuration update** to re-apply service bindings to every edge machine (slowest, finished 23:03).
3. Used database recovery methods; restored ~800 prefixes, then the remaining ~300 manually.
4. Cloudflare noted its snapshot-based rollback system "is not in Production today", and manual rollback needed database snapshots.

## Permanent fix (as listed by Cloudflare)

| Change | Pattern it addresses |
|---|---|
| **Standardise the API schema** so flags like `pending_delete` are validated (it was interpreted as a string) | Ambiguous input |
| **Separate operational from configured state**: deploy snapshots of database state through health-mediated deployments so bad states can be reverted near-instantly | Fast, safe rollback |
| **Circuit breaker** that detects changes happening "too fast or too broadly" (e.g. withdrawing or deleting BGP prefixes quickly) and stops snapshot deployment; tie it to customer-service health signals | Blast radius of automation |
| Part of "Code Orange: Fail Small": controlled rollouts of configuration, break-glass and circular-dependency fixes, well-defined failure modes | Programme-level resilience |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Scripts calling APIs with filters (`?status=`, label selectors) | Treat empty/missing filters as **errors** in destructive operations; `kubectl delete` with an empty selector deletes everything matching — validate first |
| Cleanup CronJobs (old images, snapshots, namespaces, DNS records) | Max-delete-per-run limits; dry-run reports; alerts on deletion rate (topics 40, 47) |
| Terraform/Ansible loops built from API data | Assert expected counts before acting ([Ansible 31](../../Ansible/notes/31-error-handling.md)) |
| Staging data that doesn't resemble production | Seed staging with production-shaped data; test automation paths with **no** explicit input |
| Rollback that needs database surgery | Versioned snapshots of configuration; one-click revert |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: An API that interpreted a flag with an empty value as "return all prefixes", used by a new automated deletion task with no limit on how much it could delete. Contributing: tests and staging data didn't cover the automated, no-input path; no circuit breaker for broad withdrawals.

2. **Q: How was it detected?**
   A: Cloudflare was engaged for failures on one.one.one.one about 17 minutes after withdrawals began; incident declared at 18:18.

3. **Q: What would you check first?**
   A: What's being withdrawn and by which process (Addressing API change logs, running tasks), then stop it.

4. **Q: How did they stop the impact?**
   A: Killed the sub-process, then restored prefixes via dashboard toggles, database recovery and a global configuration push for service bindings.

5. **Q: Why did some customers recover much faster than others?**
   A: Customers whose prefixes were only withdrawn could re-advertise themselves; those whose service bindings were deleted needed Cloudflare to re-apply bindings to every edge machine.

6. **Q: What changed permanently?**
   A: Strict API schemas, snapshot-based configuration deployment with fast rollback, and a circuit breaker for fast or broad withdrawals.

7. **Q: How would you confirm the fix?**
   A: Tests calling the API with empty or missing flags must fail; a simulated mass withdrawal must trip the circuit breaker.

8. **Q: What if the engineer hadn't noticed for hours?**
   A: The task would have kept deleting — potentially all 4,306 prefixes. That's why automatic rate/scope limits matter more than human vigilance.

9. **Q: How would you prevent this class of incident?**
   A: Destructive automation must require explicit, validated targets, have per-run limits, run in stages, and be easy to roll back.

10. **Q: How is this like Atlassian 2022 (topic 47)?**
    A: Both: an automated deletion received the wrong scope of targets (site IDs; "all" prefixes), and restoration of dependent objects was slower than the deletion. Soft-delete, limits and tested bulk restore are the shared lessons.

## Source

- Cloudflare, "Cloudflare outage on February 20, 2026": <https://blog.cloudflare.com/cloudflare-outage-february-20-2026/>
