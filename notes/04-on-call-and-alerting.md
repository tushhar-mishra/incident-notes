# 04 · On-Call and Alerting

> **Level:** ★ foundation
>
> **You will learn:** what **on-call** means day to day, how **rotations**, **primary/secondary** and **escalation** work, how a **page** reaches you, **symptom vs cause** alerts, what makes an alert **actionable**, how to fight **alert fatigue**, how to do a good **handoff**, and how to look after yourself on call.
>
> **You need:** topics 01–03.
>
> **Part 1 · Incident basics** · Previous: [03 · SLOs and error budgets](03-slos-and-error-budgets-for-on-call.md) · Next: [05 · Incident roles](05-incident-roles-and-commander.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **On-call** means: for a period (often a week), **you** are the person who responds when production needs a human, day or night. Alerts that need a human **now** become **pages** (phone call / push). Everything else becomes a **ticket** or nothing.

```
Monitoring ──► Alert rule fires ──► Alert manager (group, dedupe, route)
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
                 PAGE primary      TICKET (queue)       Dashboard only
                     │ no ack in 5–10 min
                     ▼
                 PAGE secondary ──► no ack ──► PAGE manager
```

**Picture it.** A doctor on call at a hospital. They don't get phoned for every patient's cough (ticket), only for real emergencies (page). If they don't answer, the hospital phones the next doctor (escalation).

**The one thing to know.** **Every page must be urgent, real, and actionable.** If you can't do anything about an alert at 3 AM, it should not wake you.

**Try it now:** list the alerts that paged you (or your team) last month. For each one, ask: did a human have to act **right then**? Delete or downgrade the ones where the answer is no.

**You've got it if you can:** explain symptom vs cause alerts and design an escalation policy.

> 📖 **That's the core.** Everything below is the **full reference**. Design-level view: [SD 29 · Alerting and on-call design](../../System-Design/notes/29-alerting-and-on-call.md).

---

## What it is

| Term | Meaning |
|---|---|
| **On-call rotation** | A schedule that rotates who is on duty (e.g. weekly, Monday 10:00 to Monday 10:00) |
| **Primary / secondary** | Primary gets paged first; secondary is backup if primary doesn't respond |
| **Escalation policy** | Rules for who gets paged next, and after how long, if nobody acknowledges |
| **Acknowledge (ack)** | Pressing "I'm on it" so the system stops escalating |
| **Follow-the-sun** | Teams in different time zones cover their own daytime, so nobody is paged at night |
| **Alert** | A rule on a metric/log/check that fires when a condition is true |
| **Page vs ticket** | Page = wake a human now. Ticket = fix in working hours |
| **Alert fatigue** | So many (often useless) alerts that people start ignoring them, including the real ones |
| **Runbook** | Instructions linked from an alert: what it means and what to do (topic 09) |

### Symptom alerts vs cause alerts

| Type | What it watches | Example | Page? |
|---|---|---|---|
| **Symptom** | What users feel | "Checkout error budget burning 14× too fast"; "p99 latency > 1 s for 10 min"; "synthetic login check failing" | **Yes** |
| **Cause** | An internal component | "CPU > 90%"; "one pod restarted"; "disk 80%" | Usually **no**: ticket or dashboard |
| **Imminent-symptom** | A cause that **will** hurt users soon and can't wait | "Disk will be full in 4 hours"; "certificate expires in 3 days" | Yes, if the time to impact is short |

Why symptoms: a cause alert can be noisy (CPU spikes are often fine) and incomplete (there are a thousand possible causes; you can't alert on all of them). A symptom alert catches **every** cause that hurts users.

---

## Why it exists

Production breaks outside working hours. Someone must be responsible, or everyone assumes someone else is. A good on-call system:

- Gets the right person **fast** (low MTTA, Mean Time To Acknowledge, topic 10).
- **Protects** people from pointless pages, because exhausted engineers make mistakes.
- Makes ownership clear: "you build it, you run it" means the team that writes a service also carries its pager.

---

## Analogy

**Firefighters on shift.** They don't run out for every smoke detector beep in a single kitchen (cause alert). They respond when a building's alarm system confirms a real fire (symptom alert). If one crew is busy, the call goes to the next station (escalation). They hand over at shift change with a briefing (handoff). They rest between calls, because a tired firefighter is dangerous.

---

## How it works

### The life of a page

1. A metric crosses the alert rule threshold (e.g. Prometheus evaluates the rule every 30 s).
2. The rule must stay true for a `for:` duration (e.g. 5 minutes) to avoid blips.
3. The **alert manager** (e.g. Prometheus Alertmanager) groups related alerts, removes duplicates, and silences known maintenance.
4. It routes by labels (`severity=page`, `team=payments`) to a paging tool (PagerDuty, Opsgenie, Grafana OnCall…).
5. The paging tool calls the **primary**. No ack in N minutes → **secondary** → manager.
6. The on-call acks, opens the **runbook link** in the alert, and starts triage (topic 07).

### What makes an alert good

A good page answers, in its text:
- **What** is wrong, in user terms ("Checkout 5xx at 4%, SLO allows 0.1%").
- **Where** (service, region, cluster).
- **Since when**, and a link to the **dashboard**.
- A **runbook** link.
- **Severity**.

Checklist for every alert rule:
- [ ] Is it **urgent**? (Would waiting until morning hurt users?)
- [ ] Is it **actionable**? (Is there something a human can do?)
- [ ] Is it **real**? (Has it been a false alarm more than once?)
- [ ] Does it have a **runbook**?
- [ ] Does it have an **owner** team?

### Handoff

At the end of a shift, write a short note to the next person:

```
On-call handoff: payments, week 46
- Open incidents: none. INC-231 (slow refunds) resolved Tue, postmortem Fri.
- Watch: DB disk on pay-db-2 at 78%, growing 2%/day. Ticket OPS-881.
- Noisy alerts: "QueueDepthHigh" fired 9× with no impact → ticket to tune.
- Changes planned: Postgres minor upgrade Thu 14:00 (change CHG-77).
```

### Healthy on-call

- **Load target:** the Google SRE book suggests an on-call engineer should handle at most about **two incidents per 12-hour shift**, so there is time to respond well and follow up. If you're over that, the system (not the person) needs fixing.
- **Compensation and time off** after a bad night.
- **Shadowing** before your first shift (watch an experienced on-call).
- **Review pages weekly:** every page that wasn't actionable gets a ticket to fix or delete the alert.

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Alert fatigue** | Dozens of pages a week, people ack and ignore | Weekly page review; delete non-actionable alerts; move cause alerts to tickets |
| **Missing symptom alerts** | Customers report outages before monitoring does | Burn-rate alerts on SLIs, synthetic checks from outside |
| **Alert on the thing that's down** | Monitoring runs in the same cluster/region it monitors, so it dies too | Out-of-band monitoring (topic 44) |
| **No escalation** | Primary's phone was on silent; nobody else paged for an hour | Escalation policy with timeouts; test it |
| **Hero culture** | One person always fixes everything, burns out, leaves | Rotations, runbooks, shadowing, shared knowledge |
| **Alert without a runbook** | On-call stares at "KafkaLagHigh" not knowing what it means | Runbook link required in every paging rule |

---

## Where you'll use it

- **Kubernetes monitoring:** Prometheus rules + Alertmanager routing from [K8s 79](../../CKA-Certification-Course-2025/notes/79-monitoring-and-logging-basics.md) and [K8s 80](../../CKA-Certification-Course-2025/notes/80-production-observability.md).
- **Docker hosts:** container restart and OOM alerts from [Docker 47 · Monitoring containers](../../Docker/notes/47-monitoring-containers.md).
- **CI/CD:** pipeline notifications are not pages; deploy failures to prod might be ([Jenkins 71 · Notifications](../../Jenkins/notes/71-notifications.md)).
- **Interviews:** "How would you reduce alert fatigue?" is a very common senior question.

---

## Quick recap

- On-call = you respond for a time window; rotations and escalation make sure someone always answers.
- Page only for **urgent, real, actionable** problems; everything else is a ticket.
- Prefer **symptom** alerts (what users feel) over **cause** alerts (CPU, one pod).
- Every page needs a runbook link and an owner.
- Hand off in writing. Review pages weekly. Fix the system, not the person.

### Interview questions

1. **Q: How would you reduce alert fatigue on a team that gets 50 pages a week?**
   A: First, measure: export every page from the last month and label each one actionable or not. Delete or downgrade non-actionable ones to tickets; merge duplicates through Alertmanager grouping; replace many cause alerts (CPU, single pod restarts) with a few symptom alerts (SLO burn rate, synthetic checks); add `for:` durations to stop flapping; make every remaining page have a runbook. Then review pages weekly so it stays fixed.

2. **Q: Should you page on high CPU?**
   A: Usually not. High CPU is a cause that may or may not hurt users. I'd page on what users feel (errors, latency) and keep CPU as a dashboard signal or a ticket. Exceptions are cases where saturation reliably predicts imminent user impact and nothing else catches it.

3. **Q: Your primary on-call didn't acknowledge a SEV1 page for 20 minutes. What went wrong and how do you fix it?**
   A: The escalation policy failed or doesn't exist. The page should escalate to the secondary after 5–10 minutes and then to a manager. I'd check whether escalation was configured, whether contact methods were correct, and whether the primary was overloaded or unreachable, then fix the policy and test it with a fake page. Blameless: people's phones die; the system should handle that.

4. **Q: What's in a good on-call handoff?**
   A: Open incidents and their state, things to watch (trends that might become incidents), noisy alerts and the tickets to fix them, and planned changes during the next shift.
