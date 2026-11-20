# 10 · Incident Metrics

> **Level:** ★★ intermediate
>
> **You will learn:** the incident timeline **timestamps**, and the metrics built from them: **MTTD** (Mean Time To Detect), **MTTA** (Mean Time To Acknowledge), **MTTM** (Mean Time To Mitigate), **MTTR** (Mean Time To Recover / Resolve / Repair — it has several meanings), **MTBF** (Mean Time Between Failures), the **DORA** metrics **change failure rate** and **failed deployment recovery time**, how to **improve** each one, and **why averages lie** about incidents.
>
> **You need:** topics 01–09.
>
> **Part 1 · Incident basics** · Previous: [09 · Runbooks](09-runbooks-and-playbooks.md) · Next: [11 · Blameless postmortems](11-blameless-postmortems.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every incident has the same key moments. The gaps between them are what you measure and improve.

```
 failure   detected   acknowledged   mitigated        resolved      next failure
 starts    (alert)    (human ack)    (users OK)       (fully fixed)
   │──TTD──►│──TTA────►│─────TTM──────►│───────────────►│ ... ─MTBF─►│
   │◄────────────────── TTR (time to recover) ─────────►│
```

- **MTTD** — Mean Time To Detect: how long before we knew.
- **MTTA** — Mean Time To Acknowledge: how long until a human responded.
- **MTTM** — Mean Time To Mitigate: how long until users were OK.
- **MTTR** — Mean Time To Recover (or Resolve / Repair / Respond — **define it**).
- **MTBF** — Mean Time Between Failures: how often things break.

**Picture it.** A house fire: how long until the alarm went off (TTD), until the fire brigade answered (TTA), until the fire was out (TTM), until the house was livable again (TTR).

**The one thing to know.** "MTTR" means different things at different companies. **Always say which one you mean**, and track **detection** and **mitigation** separately, because they are improved in different ways.

**Try it now:** for your last incident, write down the five timestamps.

**You've got it if you can:** explain how you would reduce MTTD and MTTM, and why the average of incident durations can mislead.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Metric | From → to | Improve it with |
|---|---|---|
| **MTTD** | Failure starts → detected | Symptom alerts on SLIs, burn-rate alerts, synthetic checks, testing alerts |
| **MTTA** | Alert fires → human acknowledges | Good escalation policy, reachable on-call, fewer noisy alerts (so pages are trusted) |
| **MTTM** | Detected → user impact stops | Rollback buttons, feature flags, runbooks, practised failover, "mitigate first" culture |
| **MTTR** | Start or detect → fully recovered | All of the above + good debugging tools |
| **MTBF** | One failure → next failure | Postmortem action items, safer deploys, testing, capacity planning |

### DORA metrics

**DORA** (DevOps Research and Assessment, now part of Google Cloud) studies software delivery performance. Its well-known delivery metrics include two that are about incidents:

| DORA metric | Meaning |
|---|---|
| **Change failure rate** | % of deployments that cause a failure needing remediation (rollback, hotfix, incident) |
| **Failed deployment recovery time** | How long it takes to recover from a deployment that failed (earlier reports called this "time to restore service") |
| Deployment frequency | How often you deploy |
| Lead time for changes | Commit → running in production |

The research found that the best teams deploy **more often** and **also** have **lower** change failure rates and faster recovery: speed and stability go together when changes are small and automated.

---

## Why it exists

- You can't improve what you don't measure.
- Separating the phases tells you **where** the time goes. An incident with 40 minutes to detect and 5 minutes to fix needs **better alerts**, not better runbooks.
- Leadership understands these numbers; they justify reliability work.

---

## Analogy

**A pizza delivery.** Time for the shop to notice your order (detect), for a cook to start (acknowledge), for the pizza to reach you (mitigate), for the shop to fix the oven that burned the last one (resolve). If deliveries are slow, you need to know **which** step is slow before you buy a faster scooter.

---

## How it works

### Record the timestamps

The **scribe** (topic 05) or the incident tool records them. Use **UTC**. Typical fields:

```
Impact start (from graphs, often found afterwards):  13:58
Detected (first alert or report):                     14:05
Acknowledged:                                          14:06
Incident declared:                                     14:08
Mitigated (SLI back in SLO):                           14:52
Resolved / closed:                                     15:40
```

Here: TTD = 7 min, TTA = 1 min, TTM = 47 min (from detection), total impact = 54 min.

### Why averages lie

Incident durations are **very skewed**: many short ones and a few huge ones.

```
Durations (min): 5, 6, 8, 7, 9, 6, 5, 480
Mean   = 65.75 min   ← dominated by one outage
Median = 6.5 min
```

- The **mean** jumps around a lot from quarter to quarter just because of one big outage, so "MTTR went up 40%" may mean nothing.
- Researchers and practitioners (for example the "VOID" incident database report by Verica, 2021–2022) have argued that MTTR is a poor measure of reliability for this reason.

Better practice:
- Report **median and percentiles** (p90), and the **count** of incidents by severity.
- Look at **individual big incidents** in detail (postmortems), not just the number.
- Track **SLO / error budget** consumption (topic 03): it measures what users felt.

### Using the metrics well

| Good use | Bad use |
|---|---|
| "Detection is our weak spot: 60% of SEV2s were reported by customers" → invest in alerts | Rewarding teams for low MTTR → people close incidents early or don't declare |
| Trend of change failure rate after adding canaries | Comparing teams with very different systems |
| Postmortem: "TTD was 40 min because the alert used averages" | Targets like "MTTR must be < 30 min" without context |

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Undefined MTTR** | Two teams report "MTTR" that mean different things | Define each metric in writing |
| **Gaming** | Incidents not declared, or closed too early, to look good | Measure impact via SLOs; blameless culture; no metric targets for individuals |
| **Only averages** | Wild swings, wrong conclusions | Median, p90, counts, and narrative |
| **Missing start time** | TTD unknown because impact start wasn't found | Reconstruct from graphs in the postmortem |

---

## Where you'll use it

- **Postmortems** (topic 11) include the timeline and these durations.
- **CI/CD design:** change failure rate is the main argument for canaries and automated rollback ([SD 34](../../System-Design/notes/34-deployment-strategies.md), [SD 39 · DORA](../../System-Design/notes/39-platform-engineering.md)).
- **Interviews:** "How did you measure improvement?" in your STAR answer (topic 91).

---

## Quick recap

- Timestamps: start → detect → ack → mitigate → resolve.
- MTTD, MTTA, MTTM, MTTR, MTBF; always define MTTR.
- DORA: change failure rate and failed deployment recovery time measure incident-causing changes.
- Durations are skewed: use medians, percentiles, counts and SLOs, not just means.
- Use metrics to find where time goes, not to judge people.

### Interview questions

1. **Q: What's the difference between MTTD and MTTR, and how would you improve each?**
   A: MTTD is how long a problem exists before we know; it improves with symptom-based and burn-rate alerts, synthetic checks and alert testing. MTTR is how long until service is restored; it improves with fast mitigation tools (rollback, flags, failover), good runbooks, practised responders and observability.

2. **Q: Your MTTR doubled this quarter. Is reliability worse?**
   A: Not necessarily. One very long incident can double the mean. I'd look at the median and p90, the number of incidents by severity, error budget consumption, and read the big incident's postmortem before drawing conclusions.

3. **Q: What is change failure rate and why does it matter?**
   A: It's the percentage of deployments that cause a failure requiring remediation. It shows how safe your delivery process is. Lowering it usually comes from smaller changes, better tests, canary deployments with automated analysis and feature flags.
