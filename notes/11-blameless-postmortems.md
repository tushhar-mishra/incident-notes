# 11 · Blameless Postmortems

> **Level:** ★★ intermediate
>
> **You will learn:** what a **postmortem** is and **when** to write one, what **blameless** really means (and what it doesn't), a complete **template**, how to build an accurate **timeline**, why there is rarely **one root cause** (contributing factors), the strengths and **limits of "5 whys"**, how to write **action items that get done**, how to run the **review meeting**, and a full **worked example**.
>
> **You need:** topics 01–10.
>
> **Part 1 · Incident basics** · Previous: [10 · Incident metrics](10-incident-metrics.md) · Next: [12 · The root-cause map](12-the-root-cause-map.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **postmortem** is a written review after an incident: what happened, why the system allowed it, and what we will change. **Blameless** means we assume everyone acted reasonably with what they knew, and we look for **system** weaknesses (tools, process, design), not someone to punish.

```
Summary → Impact → Timeline → Contributing factors → What went well / badly / lucky
→ Action items (owner + due date): PREVENT · DETECT · MITIGATE
```

**Picture it.** Aviation accident investigations. The goal is not to fire the pilot; it's to change the cockpit, the checklist or the training so **no** pilot can make that mistake again. That's why flying is so safe.

**The one thing to know.** **"Human error" is never the root cause.** It's the starting point of the question: *why was it so easy to make that mistake, and why did it do so much damage?*

**Try it now:** take any outage from Part 3 and list three contributing factors that are **not** "someone made a mistake".

**You've got it if you can:** write a postmortem with at least three contributing factors and specific action items in each category.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### When to write one

Common triggers (set your own):
- Any **SEV1 or SEV2**.
- Any **data loss**, security incident, or SLA breach.
- An incident that used a large part of the **error budget** (e.g. > 20% of the month).
- A **near miss** that could have been a SEV1.
- Anyone on the team asks for one.

### Blameless: what it means

| Blameless **is** | Blameless is **not** |
|---|---|
| Assuming people acted reasonably given what they knew at the time | Pretending nobody did anything |
| Asking "how did the system let this happen?" | Ignoring repeated, deliberate negligence (that's a management issue, handled separately) |
| Naming actions and decisions honestly in the timeline | Hiding who did what |
| Fixing tools and processes | "Be more careful next time" |

Why it matters: if people are punished for honest mistakes, they **hide information**. The timeline becomes wrong, the real causes stay hidden, and the incident repeats.

**Hindsight bias:** after the fact, the right action looks obvious. At the time, the engineer saw a dashboard with 40 alerts and two minutes to decide. Write the timeline from **their** point of view: what did they see, what did they know?

---

## Why it exists

- **Stop repeats.** The same class of failure should not hurt you twice.
- **Share learning.** Other teams learn from your incident for free.
- **Build trust** with customers (public postmortems, like many in Part 3).
- **Justify investment.** "This class of incident cost us 3 SEV1s this year" gets reliability work prioritised.

---

## Analogy

**Aviation safety boards** (Core). Also: a football team watching the replay of a lost match. They don't shout at the goalkeeper; they look at why the defence left a gap, and they practise that situation in training.

---

## How it works

### The template

```markdown
# Postmortem: <title>    (INC-1234)
Date: 2026-11-20   Severity: SEV1   Status: Draft / Reviewed / Actions complete
Authors: …   Incident Commander: …

## Summary (3–4 lines)
What happened, impact, duration, main contributing factors, how it was resolved.

## Impact
- Users affected: ~25% of checkouts, all regions
- Duration: 13:58–14:52 UTC (54 min); degraded until 15:40
- Error budget: 62% of November's checkout budget
- Revenue / SLA / data: est. N failed orders; no data loss; no SLA breach

## Timeline (UTC)
13:58  payments-svc v2.14 deployed to 100% (pipeline #812)
14:00  provider status page: "degraded performance"
14:05  Alert CheckoutErrorBudgetFastBurn fires; on-call ack 14:06
14:08  SEV2 declared; IC: Priya
14:10  Upgraded to SEV1
14:12  Decision: roll back v2.14
14:20  Rollback complete; errors 25% → 6%
14:28  Secondary provider enabled for 50% of traffic; errors < 0.5% by 14:52
15:40  Provider recovered; incident resolved

## Contributing factors
1. New retry logic retried 3× with no backoff or retry budget, tripling load on a
   degraded provider.
2. Deploy went to 100% at once; no canary analysis on the checkout SLI.
3. No alert on provider latency; we learned of the degradation from their status page.
4. Failover to the secondary provider was manual and undocumented (took 8 min to find).

## What went well / what went badly / where we got lucky
+ Rollback took 8 minutes and worked first time.
– Status page updated 20 min after declaration.
? Lucky: the provider recovered on its own; we had no plan B for a full provider outage.

## Action items
| # | Type     | Action                                                     | Owner | Due    | Ticket  |
|---|----------|------------------------------------------------------------|-------|--------|---------|
| 1 | Prevent  | Shared retry library: exponential backoff + jitter + budget | Sam   | Dec 10 | PAY-901 |
| 2 | Prevent  | Canary (5%) with automated analysis + rollback on SLI      | Lee   | Dec 15 | PLAT-33 |
| 3 | Detect   | Per-provider latency and error alerts                       | Ana   | Nov 30 | PAY-902 |
| 4 | Mitigate | Automatic provider failover via circuit breaker; runbook     | Sam   | Jan 15 | PAY-903 |
```

### Building the timeline

Sources: chat channel, alert history, deploy logs, audit logs (CloudTrail, Kubernetes audit, Git), graphs. The **scribe's** notes are the backbone. Add:
- **Impact start** (often earlier than the first alert; read it from the graph).
- **Decisions** and **why** they were made ("we rolled back because the deploy lined up with the start").

### Root cause vs contributing factors

Complex systems rarely fail for **one** reason. They fail when several defences have holes at the same time. This is often called the **Swiss cheese model** (James Reason): each defence layer is a slice with holes; an accident happens when the holes line up.

```
  Code review     Tests        Canary        Alerting      Rollback
  ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐
  │  ○   │      │    ○ │      │ ○    │      │  ○   │      │      │
──┼──○───┼──────┼──○───┼──────┼──○───┼──────┼──○───┼──────┼──●   │  ← stopped here
  │      │      │      │      │      │      │      │      │      │
  └──────┘      └──────┘      └──────┘      └──────┘      └──────┘
```

So write **several** contributing factors: the **trigger**, the **latent weakness** (bug, missing limit), and the **failed defences** (why tests, canary, alerting didn't catch it).

### 5 whys, and its limits

**5 whys** = keep asking "why?" to go from symptom to deeper cause.

```
Checkout failed.                     Why? → The provider calls timed out.
Why?  → We sent 3× normal traffic.   Why? → New code retried 3× with no backoff.
Why?  → No shared retry library; each team writes its own.
Why?  → Nobody owns client resilience standards.
```

**Limits:** it produces **one chain**, but incidents have **many** causes; it can stop at a person ("why? because Bob forgot"); different people get different answers. Use it as a **tool to go deeper**, and branch it into several chains.

### Action items that get done

| Bad | Good |
|---|---|
| "Be more careful with deploys" | "Add 5% canary with automated SLI analysis to payments pipeline (Lee, Dec 15, PLAT-33)" |
| "Improve monitoring" | "Alert on per-provider error rate > 2% for 5 min, with runbook (Ana, Nov 30)" |
| "Fix the bug" | "Fix the bug **and** add a regression test **and** a config size limit" |
| 25 action items | 3–6 high-leverage items, prioritised |

Categories:
- **Prevent:** stop this class of failure (validation, guardrails, limits, design changes).
- **Detect:** find it faster (alerts, checks).
- **Mitigate:** reduce impact / recover faster (rollback, failover, runbooks, smaller blast radius).

Follow-through: tickets in the team backlog, a weekly review of open items, completion rate reported to leadership, and error budget policy giving them priority.

### The review meeting

- Within about **5 business days**, while memory is fresh.
- Draft written in advance; meeting is for **discussion and agreement**, not reading.
- A facilitator (not the IC) keeps it blameless: redirect "why didn't you…" to "what made that hard to see?".
- Publish widely internally. For customer-facing incidents, consider a public version (most outages in Part 3 come from public postmortems).

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Blame** | "Root cause: engineer error" | Facilitator; ask what made the error easy and impactful |
| **Single root cause** | One-line cause, one fix, repeats anyway | Contributing factors; Swiss cheese |
| **Vague actions** | "Improve testing" never happens | Specific, owned, dated, ticketed |
| **Too many actions** | 30 items, none finished | Prioritise 3–6 high-leverage ones |
| **Never written** | "We were too busy" | Required for SEV1/2; template; time allocated |
| **Not shared** | Another team repeats the same mistake | Publish internally; monthly incident review |

---

## Where you'll use it

- **Every Part 3 outage** is a public postmortem; read them as examples of good (and sometimes less good) practice.
- **System Design:** [SD 30](../../System-Design/notes/30-incident-management.md) and [SD 44 · Chaos engineering](../../System-Design/notes/44-chaos-engineering-and-game-days.md).
- **Interview:** the "Result" part of a STAR story is often your postmortem actions (topic 91).

---

## Quick recap

- Postmortem = written review: summary, impact, timeline, contributing factors, actions.
- Blameless = look for system weaknesses; people acted reasonably with what they knew.
- "Human error" is a starting point, not a root cause.
- Several contributing factors (Swiss cheese). 5 whys is a tool, not the answer.
- Actions: specific, owned, dated; prevent / detect / mitigate; tracked to done.

### Interview questions

1. **Q: What makes a postmortem blameless, and why does it matter?**
   A: It assumes people acted reasonably with the information they had and looks for how the system — tools, processes, design, incentives — made the mistake possible and impactful. It matters because blame makes people hide information; without honest timelines you fix the wrong things and incidents repeat.

2. **Q: An engineer ran a command against production instead of staging. What's the root cause?**
   A: Not "the engineer". I'd ask: why could one command reach production that easily? Were environments visually distinct? Did the tool require confirmation or a dry run? Did the engineer need production credentials for that task at all? Was there a guardrail on the blast radius? Actions would be things like separate credentials per environment, prompts and colours, least privilege, and limits in the tool.

3. **Q: What are the limits of "5 whys"?**
   A: It produces a single linear chain, while real incidents have several interacting causes; it often stops at a person; and different investigators get different answers. I use it to dig deeper, but branch it and combine it with timeline analysis and a list of contributing factors.

4. **Q: How do you make sure action items get done?**
   A: Few and specific, each with an owner, due date and ticket; prioritised by the error budget policy; reviewed weekly; and completion rates reported. I also prefer automated guardrails over process items, because "be careful" decays and a guardrail doesn't.
