# 47 · Atlassian, April 2022: The Script Got Site IDs

> **Level:** ★★ intermediate
>
> **Pattern:** Automation given the **wrong kind of ID** + an API that deletes whatever ID it gets + restore built for one site, not hundreds. **Families:** human error/tooling · automation gone wrong · backups/restore at scale.
>
> **You will learn:** how a planned cleanup of a retired app deleted **883 entire cloud sites (775 customers)** in 23 minutes, why monitoring didn't notice (the deletions used a "standard workflow"), why restoring took **up to 14 days** although backups worked, what "**Restoration 1 vs Restoration 2**" was, and Atlassian's fixes: **universal soft deletes** and **multi-site restore automation**.
>
> **You need:** Part 1, topics 34, 40.
>
> **Part 3 · Real outages · Human error and tooling** · Previous: [46 · GitLab 2017](46-gitlab-2017.md) · Next: [48 · Google Cloud × UniSuper 2024](48-google-cloud-unisuper-2024.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A team needed to delete a retired app ("Insight – Asset Management") from customer sites. They asked another team to run a deletion script. The request contained the **IDs of whole cloud sites**, not the IDs of the app. The API accepted **either** type and deleted **whatever** it was given, with no warning. 883 sites were deleted.

```
Request: "delete the app on these sites" → list contains SITE IDs (should be APP IDs)
Script → deletion API: "site ID? delete the site.  app ID? delete the app."  (no check, no warning)
07:38–08:01 UTC: 883 sites deleted (775 customers) → Jira, Confluence, Opsgenie, Statuspage gone for them
Backups: fine (≤ 5 min data loss)  but  restore process: built for ONE site at a time → up to 14 days
```

**Picture it.** You ask the removals company to "take away the old sofa from these flats", but you hand them a list of **flat numbers**, not sofa labels. Their instructions say "remove whatever is on the list". They empty 883 flats.

**The one thing to know.** Two lessons: **destructive APIs must not accept ambiguous input** (and should soft-delete), and **you must be able to restore at the scale you can delete**.

**You've got it if you can:** explain why backups worked but recovery still took two weeks.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 5 April 2022, a maintenance script meant to delete a legacy app from customer sites was given the IDs of the **sites themselves**. The deletion API accepted site IDs and permanently deleted 883 sites belonging to 775 customers. Atlassian restored all of them by 18 April with no more than five minutes of data loss per customer, but restoration took up to 14 days because their tooling could restore single sites, not hundreds of interconnected multi-product sites at once.

## Impact

| Item | Detail (from Atlassian's PIR) |
|---|---|
| Customers / sites | **775 customers, 883 sites** |
| Products unavailable | Jira family, Confluence, Atlassian Access, Opsgenie, Statuspage |
| Duration | **Up to 14 days** for some customers; first restored 8 April; all restored by **18 April** |
| Data loss | "no customer lost more than five minutes of data" (RPO met; RTO missed) |
| Unaffected | Over 99.6% of customers and users had no disruption |
| Security | Not a cyberattack; no unauthorised access to customer data |
| Side effect | Some customer **contact information was deleted** too, so affected customers struggled to reach support |
| Cost | **Not publicly disclosed** |

## Timeline (UTC, from Atlassian's PIR)

| Time | Event |
|---|---|
| Apr 5 07:38 | Script starts |
| 07:38–08:01 | 883 sites deleted (sequentially from the input list) |
| 07:46 | First support ticket from an affected customer |
| 08:17 | Major incident process triggered |
| 08:24 | Escalated to Critical |
| 08:53 | Team confirms the tickets relate to the script run (cause identified within an hour of the script finishing) |
| 12:38 | Highest severity, once the complexity of restoration is understood |
| Apr 8 03:30 | Company-wide code freeze |
| Apr 8 | First customers restored |
| Apr 9 | "Restoration 2" approach proposed |
| Apr 18 | All customers restored |

## How it happened

```
Team A (app retirement)  ──"delete the app for these customers"──►  Team B (runs scripts)
        │                                                               │
        └─ list contains SITE IDs instead of APP IDs ─────────────────►│
                                                                        ▼
                                       deletion API: accepts site IDs OR app IDs
                                       - app ID  → deletes the app
                                       - site ID → deletes the WHOLE SITE
                                       - no warning about which type it got
                                                                        │
                                Run 1: 30 sites with correct app IDs ✅ │
                                Run 2: 883 SITE IDs ───────────────────►  883 sites permanently deleted
```

Atlassian's contributing factors, in their words:
1. **Communication gap:** "Instead of providing the IDs of the intended app being marked for deletion, the team provided the IDs of the entire cloud site."
2. **Insufficient system warnings:** "There was no warning signal to confirm the type of deletion (site or app) being requested."
3. **Review checked the wrong thing:** peer review focused on which endpoint was called and how; it "did not cross-check the provided cloud site IDs".
4. **Testing couldn't catch it:** the IDs didn't exist in staging.

## Why it spread

- **Batch execution** with no limits: one run touched 883 sites in 23 minutes.
- **Permanent deletion** instead of soft delete.
- **Restore designed for single sites:** Atlassian: "We had the ability to recover a single site, but we had not built capabilities and processes for recovering a large batch of sites."
- **Interconnected products:** each site's Jira, Confluence, etc. had to be restored and re-linked across many systems.
- **Contact data deleted** → communication with affected customers was harder.

## How they found it

- **Customers**, not monitoring: the first support ticket arrived **8 minutes** after the script started. Atlassian notes the sites "were deleted via a standard workflow", so monitoring did not flag it.
- Support tickets were correlated with the script run by 08:53.

## What they did to fix it

1. Major incident process, cross-functional teams, a Jira project ("SITE") tracking each site's restoration, team syncs every three hours, code freeze.
2. **Restoration 1** (~70 mostly sequential steps): create **new** sites (new `cloudId`s) and re-map data. Once optimised, about 48 hours per batch; it restored 53% of affected users across 112 sites. Atlassian estimated it would have taken three weeks to restore everyone this way.
3. **Restoration 2** (proposed 9 April): **un-delete** records and **reuse the original site IDs**, removing over half the steps and avoiding problems for third-party apps that rely on the IDs. Ran in parallel while being tested, then used for the remaining sites.
4. All customers restored by 18 April.

## Permanent fix (Atlassian's four main action items)

| Action | Pattern it addresses |
|---|---|
| **Universal soft deletes** across all systems; deletions prohibited or with multiple layers of protection, staged rollout and tested rollback; no deletion of customer data/metadata without soft delete | Irreversible destructive operations |
| **Accelerate the DR programme**: automate restoration for multi-site, multi-product deletion; run regular DR exercises restoring many sites | "Restore at the scale you can delete" |
| **Revise incident management for large-scale incidents** (simulations, tooling for many parallel teams) | Coordination |
| **Large-scale incident communications playbook**: acknowledge early, multiple channels, backup of key contacts, support access without a valid site/ID | Communication (topic 06) |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| A script that takes a list of IDs and calls delete APIs | Validate the **type** and **count** of IDs; dry-run printing names, not IDs; refuse more than N per run; soft-delete first |
| `kubectl delete -f list.yaml` / `xargs kubectl delete ns` from a generated list | Review the generated list; labels + admission policies protecting critical namespaces |
| Ansible playbook run against a dynamic inventory group that unexpectedly contains production | `--list-hosts` before running; guard tasks; `serial` ([Ansible 42](../../Ansible/notes/42-dynamic-inventory.md)) |
| Terraform `for_each` over a map built from data → mass destroy | Plan review; destroy-count policy ([Terraform 54](../../Terraform/notes/54-policy-as-code.md)) |
| Backups exist but restore is manual, per item | Automated bulk restore, tested in drills (topic 34) |
| Monitoring doesn't alert on **legitimate-looking** mass deletions | Alert on deletion **rate** (e.g. CloudTrail/K8s audit deletes per minute) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: The trigger was a list of wrong IDs. The root causes were an API that permanently deleted whatever ID type it received without warning, review that didn't check the IDs, no soft delete, and restore tooling that only worked one site at a time.

2. **Q: How was it detected? Why not earlier?**
   A: From customer tickets, 8 minutes after the start. Monitoring saw normal-looking deletions via a standard workflow; there was no alert on unusual deletion volume.

3. **Q: What was checked first?**
   A: Correlating incoming tickets with recent operations, which tied them to the script within about an hour of it finishing.

4. **Q: How did they stop impact quickly?**
   A: The deletion was already complete in 23 minutes; the work was restoration, first with Restoration 1, then the faster Restoration 2.

5. **Q: What did they change permanently?**
   A: Universal soft deletes, automated multi-site restore with regular DR exercises, a large-scale incident process and a communications playbook.

6. **Q: How would you confirm the fix?**
   A: A DR exercise that deletes and restores many sites end-to-end within the RTO, plus tests that the delete API rejects ambiguous IDs.

7. **Q: What if Restoration 2 had failed?**
   A: Continue Restoration 1 in parallel (they ran both); it was slower but proven.

8. **Q: How would you prevent this class of incident?**
   A: Type-safe destructive APIs, soft delete with delayed purge, blast-radius limits per run, and staged execution (a few, then more) with human checks of real names.

9. **Q: What would you do differently in communication?**
   A: Acknowledge broadly and early on all channels and keep a backup of customer contacts outside the system that could be deleted.

10. **Q: What if the same script ran against 10× more sites?**
    A: Without blast-radius limits, damage scales linearly and restoration time would be months. A per-run cap and soft delete make the size of the input much less dangerous.

## Source

- Atlassian, "Post-Incident Review on the Atlassian April 2022 outage": <https://www.atlassian.com/engineering/post-incident-review-april-2022-outage>
