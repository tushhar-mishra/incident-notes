# 53 · Google Cloud, June 2025: A Null Pointer Replicated Worldwide

> **Level:** ★★★ advanced
>
> **Pattern:** New code path with **no error handling** and **no feature flag** + **globally replicated policy data** with blank fields + **restart herd** with no randomized backoff. **Families:** change (code + data push) · overload during recovery (topic 36) · monitoring/status that shares fate (topic 44).
>
> **You will learn:** how a quota-policy feature added to Google's **Service Control** (the system that checks API requests for authorization, policy and quota) on 29 May sat dormant until a policy with **blank fields** was written on 12 June, crashing Service Control **in every region** and causing 503s across Google Cloud and Workspace APIs; why **us-central1** took much longer (a **herd effect** on Spanner); and Google's commitments: **feature flags by default, fail open, randomized exponential backoff**.
>
> **You need:** Part 1, topics 24, 36, 38, 44.
>
> **Part 3 · Real outages · Bad deploys and config pushes** · Previous: [52 · CrowdStrike 2024](52-crowdstrike-2024.md) · Next: [54 · Azure Front Door 2025](54-azure-front-door-2025.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every Google Cloud API call passes through **Service Control**, which checks policies and quotas. On 29 May 2025, new code for extra quota-policy checks was rolled out region by region. It looked fine, because the new code path only ran when a certain kind of **policy** existed. On 12 June, such a policy was written — with **blank fields** — and **replicated globally within seconds**. Service Control hit a **null pointer** and crashed everywhere, repeatedly.

```
May 29: new code rolled out (no feature flag, no error handling on this path) → dormant
Jun 12 ~10:45 PDT: policy with blank fields written to regional Spanner tables
   → replicated globally in seconds → every region's Service Control reads it
   → null pointer → crash loop → API requests get 503 across Google Cloud & Workspace
   → SRE triaging within 2 min, root cause within 10 min, "red button" disables the path (~40 min)
   → most regions recover; us-central1: restarting tasks overload Spanner ("herd"), no randomized backoff
     → takes up to ~2h40m there
```

**Picture it.** Every airport's security scanner gets a software update in May that's never used. In June, head office sends every airport a new rule with an empty box. Every scanner crashes on reading the empty box, and all flights stop. When the scanners restart, they all call head office at the same moment, and head office's phone line in one city collapses.

**The one thing to know.** A **staged rollout of code** doesn't protect you if the code is only **triggered by data** that arrives **globally at once**. Put new code behind a **feature flag**, handle errors, and **fail open** where safe.

**You've got it if you can:** explain why the region-by-region rollout didn't catch it and why us-central1 was slower.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 12 June 2025, a policy change containing unintended blank fields was inserted into the Spanner tables used by Google's Service Control and replicated globally within seconds. It exercised a new quota-policy code path (added 29 May) that had no error handling and no feature flag, causing a null pointer crash loop in Service Control in every region and 503 errors for Google Cloud, Workspace and Security Operations APIs. Most regions recovered within about two hours; us-central1 took longer because restarting tasks overloaded the underlying Spanner table.

## Impact

| Item | Detail (from Google's incident report) |
|---|---|
| Products | Google Cloud, Google Workspace and Google Security Operations: increased **503 errors** on external API requests (the status page lists around 100 products, e.g. Compute Engine, Cloud Storage, BigQuery, IAM, Gmail, Drive, Meet) |
| Not impacted | "Existing streaming and IaaS resources were not impacted" |
| Region | Global |
| Duration | Mini report: start 10:49, end 13:49 PDT, "Duration: 3 hours"; us-central1 took "up to ~2h 40 mins" to fully resolve; some products had residual impact (backlogs) longer, with the last product recovered at 18:18 PDT |
| Customers / cost | **Not publicly disclosed** |

## Timeline (PDT, 2025)

| Time | Event |
|---|---|
| May 29 | New feature added to Service Control for additional quota policy checks; rolled out region by region; the new path not exercised (needs a specific policy) |
| Jun 12 ~10:45 | Policy change with unintended blank fields inserted into regional Spanner tables; replicated globally within seconds |
| 10:49–10:51 | Incident starts (report gives both times): Service Control binaries crash-loop |
| +2 min | SRE team triaging |
| +10 min | Root cause identified |
| +~25 min | "Red-button" (switch to turn off that policy serving path) ready |
| +~40 min | Red-button rollout complete; regions begin recovering |
| 11:46 | First public status post (about an hour in; the Cloud Service Health infrastructure was itself affected) |
| 12:48 | All regions except us-central1 mitigated (mini report) |
| ~13:30 | us-central1 fully resolved ("up to ~2h 40 mins") |
| 18:18 | Last affected product (Vertex AI Online Prediction) recovered |

## How it happened

```
            ┌───────── policy data (Spanner, replicated globally in seconds) ─────────┐
            │  new policy row with BLANK fields                                         │
            ▼                ▼                ▼                ▼                ▼
   Service Control   Service Control   Service Control  ...  (every region)
   new quota-check code path (since May 29):
     - no error handling for blank fields
     - not behind a feature flag
       → null pointer → binary crashes → restarts → reads policy again → crashes (loop)
            │
   every Google API request needing Service Control → 503
```

Google's words: the change "did not have appropriate error handling nor was it feature flag protected", and "If this had been flag protected, the issue would have been caught in staging." The code path "was never exercised during this rollout due to needing a policy change that would trigger the code."

The report does **not** say how the blank fields got into the policy (**not publicly disclosed**).

## Why it spread

- **Global data, regional code:** the code rollout was staged by region, but the **policy data** was global and instantaneous, so every region hit the bug at once.
- **No fail-open:** Service Control is on the path of almost every API request; when it crashed, requests failed rather than being allowed through.
- **Recovery herd (us-central1):** "as Service Control tasks restarted, it created a herd effect on the underlying infrastructure it depends on (i.e. that Spanner table), overloading the infrastructure. Service Control did not have the appropriate randomized exponential backoff implemented to avoid this." (topic 36)
- **Status and monitoring shared fate:** the first status post came about an hour after the crashes because Cloud Service Health was affected, and some customers' monitoring running on Google Cloud also failed (topic 44).

## How they found it

SRE was triaging within **2 minutes** and identified the root cause within **10 minutes**. The report does not say which alert or signal fired (**not publicly disclosed**).

## What they did to fix it

1. **Red button:** the new code shipped with a switch to turn off that policy-serving path. Ready ~25 minutes after the start; rollout complete in ~40 minutes.
2. **us-central1:** throttled task creation to reduce load on Spanner and routed traffic to multi-regional databases.
3. Residual product backlogs recovered over the following hours.
4. **Froze all changes** to the Service Control stack and manual policy pushes until remediation is complete.

## Permanent fix (as listed by Google)

| Commitment | Pattern it addresses |
|---|---|
| **Modularize Service Control** so functionality is isolated and **fails open** | Blast radius; fail-safe design |
| **Audit all systems that consume globally replicated data**; propagate incrementally with time to validate and detect issues | Global data pushes (topic 24) |
| **All changes to critical binaries feature-flag protected and disabled by default** | Dormant code paths |
| Improve static analysis and testing to handle errors and, if needed, fail open | Error handling |
| Ensure systems use **randomized exponential backoff** | Restart herds (topic 36) |
| Improve automated and human external communications | Comms |
| Keep monitoring and communication infrastructure working even when Google Cloud and primary monitoring are down | Shared fate (topic 44) |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| New code that only runs when a certain config/flag/data appears | Feature flag **off by default**, enable per region/tenant; tests with blank/null/malformed data |
| An API gateway / auth sidecar / OPA policy check on every request | Decide fail-open vs fail-closed per check; timeouts; keep last good policy on parse errors |
| Global config (Consul KV, a ConfigMap replicated to all clusters, a feature flag service) | Staged propagation by cluster/region with health checks |
| Pods restarting together after a fix and hammering the DB | Startup jitter, backoff in clients, readiness gates, slow-start ([K8s 22 · Probes](../../CKA-Certification-Course-2025/notes/22-health-probes.md)) |
| Your monitoring in the same cloud that's down | External monitoring and status page (topics 06, 44) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A new code path with no error handling and no feature flag, which crashed on blank fields in policy data, combined with policy data that replicated globally at once. The blank-field policy was the trigger.

2. **Q: Why didn't the region-by-region rollout catch it?**
   A: The new code only ran when a matching policy existed. During the rollout no such policy existed, so the code was never exercised. The trigger came later, as global data, hitting every region simultaneously.

3. **Q: What did they check first?**
   A: The report says SRE triaged within 2 minutes and found the root cause in 10; the crash loop in Service Control pointed at the new policy path.

4. **Q: How did they stop the impact?**
   A: With the built-in "red button" that disabled the policy-serving path, then throttling restarts in us-central1.

5. **Q: What changed permanently?**
   A: Fail-open modular design, feature flags off by default for critical binaries, audits of globally replicated data, better error handling, randomized exponential backoff, and resilient communications.

6. **Q: How would you confirm the fix?**
   A: Feed blank/malformed policies in staging and show the binary logs and continues (fails open); verify restarts are jittered and don't overload the database.

7. **Q: What if the red button hadn't existed?**
   A: They'd need to roll back the binary in every region — slower — or delete the bad policy (which might also be hard if the control plane depended on Service Control). Build kill switches with the feature.

8. **Q: How would you prevent this class of incident?**
   A: Treat data that changes code behaviour as a deploy (staged, validated), handle errors defensively, and put new paths behind flags.

9. **Q: What would you do differently in recovery?**
   A: Have randomized exponential backoff and admission control on restarts from the start, so recovery doesn't create a second overload.

10. **Q: What if your company's monitoring and status page ran on Google Cloud that day?**
    A: You'd be blind and silent. Keep an external synthetic check and an independently hosted status page.

## Source

- Google Cloud Service Health, incident ow5i3PPK96RduMcb1SsW, "Multiple GCP products are experiencing Service issues" (12 June 2025), including the Mini Incident Report and full Incident Report: <https://status.cloud.google.com/incidents/ow5i3PPK96RduMcb1SsW>
