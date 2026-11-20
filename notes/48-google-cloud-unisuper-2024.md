# 48 · Google Cloud × UniSuper, May 2024: The Blank Parameter

> **Level:** ★★ intermediate
>
> **Pattern:** An internal tool with a **hidden default** + automation that **deletes on schedule**. **Families:** human error/tooling · automation gone wrong · backups (what saved the customer).
>
> **You will learn:** how one input parameter left **blank** in an internal provisioning tool gave a customer's private cloud an unknown **one-year term**, after which the system **deleted it automatically**, why the customer recovered (backups in Cloud Storage **plus third-party backup software**), and what Google changed.
>
> **You need:** Part 1, topics 34, 40.
>
> **Part 3 · Real outages · Human error and tooling** · Previous: [47 · Atlassian 2022](47-atlassian-2022.md) · Next: [49 · Knight Capital 2012](49-knight-capital-2012.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** In early 2023, Google operators used an internal tool to set up a **Google Cloud VMware Engine (GCVE)** private cloud for the Australian pension fund **UniSuper**. One **input parameter was left blank**. The system silently filled in a **default one-year term**. A year later, the system did exactly what the term said: it **deleted the private cloud**. No notification was sent, because the deletion wasn't a customer request.

```
Early 2023: operator provisions private cloud with internal tool, one parameter blank
     → system assigns a then-unknown DEFAULT: fixed 1-year term
May 2024: term ends → automatic deletion of the customer's private cloud (across two zones)
     → recovery over several days using backups in GCS + third-party backup software
```

**Picture it.** Renting a storage unit where the form had an empty "rental period" box. The office quietly wrote "1 year". Twelve months later, the unit was emptied on schedule. Luckily, you'd kept copies of your important papers elsewhere.

**The one thing to know.** **Blank should never mean "pick a dangerous default".** Required inputs must be required. And the customer's **independent backups** were what made recovery possible.

**You've got it if you can:** explain the chain from blank input to deletion, and what the customer did right.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

In early 2023, Google operators used an internal tool to deploy one of UniSuper's GCVE private clouds. One input parameter was left blank, and the system assigned a then-unknown default fixed term of one year. When that term ended, the private cloud was deleted automatically. Google and the customer worked around the clock for several days to restore it, using the customer's backups in Google Cloud Storage and third-party backup software.

## Impact

| Item | Detail (from Google's post) |
|---|---|
| Affected | **One customer** (UniSuper), one cloud region, one service (GCVE), one of the customer's GCVE private clouds (across two zones) |
| Not affected | Other Google Cloud services and customers; the customer's other private clouds, accounts and projects; their **backups in Google Cloud Storage** in the same region |
| Duration | Google says they "worked 24x7 over several days to recover"; exact outage length **not stated in Google's post** |
| Data loss | **Not stated** in Google's post |
| Cost / compensation | **Not publicly disclosed** |

## Timeline

| When | Event |
|---|---|
| Early 2023 | Private cloud deployed with an internal tool; one parameter left blank → default 1-year term |
| Q4 2023 | The internal tool deprecated and the process fully automated (customer-controlled) — but this existing deployment still carried the term |
| May 2024 | Term ends → private cloud deleted (exact date **not stated** in Google's post) |
| Following days | 24×7 joint recovery: rebuild the private cloud, restore network and security configuration, applications and data |
| 25 May 2024 | Google publishes details |

## How it happened

```
Operator ──► internal provisioning tool
              parameter X = (blank)
                   │
                   ▼
            system: "blank? use default"  →  fixed TERM = 1 year   (operators didn't know this default)
                   │
                   ▼  ... 12 months later ...
            lifecycle automation: "term expired → delete private cloud"
                   │       (no notification: not a customer-initiated deletion)
                   ▼
            customer's GCVE private cloud deleted
```

Google says operators followed internal control protocols; the danger was the **unknown default** and the **automatic deletion** tied to it.

## Why it spread

- **Automation acted exactly as configured**, at full scope: the whole private cloud, across two zones (zonal redundancy doesn't protect against deletion).
- **Notification safeguards** applied to customer-initiated deletions, not to this system-triggered one.
- **Time delay:** the cause (blank input) was a year before the effect, so nobody connected them in advance.

## How they found it

**Not publicly disclosed** in Google's post (it doesn't say how or by whom the deletion was detected).

## What they did to fix it

1. Recreate the private cloud.
2. Restore network and security configurations.
3. Restore applications and **recover data** from backups: Google says the backups stored in Google Cloud Storage "along with third party backup software, were instrumental in aiding the rapid restoration".
4. Google publicly acknowledged the incident in a joint statement with the customer.

## Permanent fix (as listed by Google)

| Change | Pattern it addresses |
|---|---|
| **Deprecated the internal tool** that triggered the events (now fully automated and customer-controlled via the UI) | Risky manual/internal paths |
| **Scrubbed the system database and manually reviewed all GCVE private clouds** to ensure no other deployments were at risk | Find every other instance of the same latent problem |
| **Corrected the system behaviour** that set private clouds for deletion in such workflows | Dangerous automatic deletion |
| Confirmed existing safeguards (soft delete, advance notification, human-in-the-loop) remain in place | Defence in depth |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Terraform variable with a dangerous default (`retention_days = 1`, `deletion_window = 7`) | No defaults for dangerous values; `validation {}` blocks; required variables ([Terraform 14](../../Terraform/notes/14-input-variables.md), [Terraform 25](../../Terraform/notes/25-conditions-and-checks.md)) |
| Helm chart value that silently enables TTL cleanup | Explicit values per environment; review rendered manifests |
| S3 lifecycle rule or Kubernetes `ttlSecondsAfterFinished`/cleanup CronJob deleting the wrong things | Scope by tag/prefix; dry-run reports; alerts on mass deletions |
| Ansible role variable left empty → default behaviour | `assert` tasks for required vars ([Ansible 31](../../Ansible/notes/31-error-handling.md)) |
| All backups inside the same account/provider as production | Independent backups (other account/provider), tested restores (topic 34) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A tool that accepted a blank required parameter and silently applied an unknown default lifetime, combined with automation that deleted the resource when that lifetime ended, without the notification path used for customer deletions.

2. **Q: How was it detected? Why not earlier?**
   A: Not disclosed. The cause was invisible for a year because the default term wasn't known to operators and nothing reported upcoming system-triggered deletions.

3. **Q: What would you check first?**
   A: What triggered the deletion (customer action, automation, attacker) using audit logs, then what backups exist outside the deleted scope.

4. **Q: How was impact stopped?**
   A: The deletion was complete; the work was recovery, made possible by the customer's backups and third-party backup tooling.

5. **Q: What changed permanently?**
   A: The internal tool was deprecated, all private clouds were reviewed for the same risk, and the deletion behaviour for such workflows was corrected.

6. **Q: How would you confirm the fix?**
   A: Query all resources for unexpected expiry/term settings, and test that a blank parameter is rejected rather than defaulted.

7. **Q: What if the customer hadn't had independent backups?**
   A: Recovery would depend entirely on the provider's ability to undelete — possibly impossible. This case is the strongest argument for backups outside the blast radius, including another provider for critical data.

8. **Q: How would you prevent this class of incident?**
   A: Required parameters fail when empty; defaults must be safe ("never expire" rather than "1 year"); scheduled deletions of production resources need notice and human approval.

9. **Q: What would you do differently as a customer?**
   A: Exactly what UniSuper's architecture allowed: keep backups the provider's automation can't delete, and practise full rebuilds.

10. **Q: What if it had happened to 1,000 customers at once?**
    A: Google's review of all private clouds was about finding exactly that: the same latent setting elsewhere. Any "time bomb" setting must be searched for across the whole fleet once one is found.

## Source

- Google Cloud, "Sharing details on a recent incident impacting one of our customers" (25 May 2024): <https://cloud.google.com/blog/products/infrastructure/details-of-google-cloud-gcve-incident>
