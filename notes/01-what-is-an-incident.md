# 01 · What Is an Incident

> **Level:** ★ foundation
>
> **You will learn:** what an **incident** is (and what it is not), the difference between an **alert**, an **incident**, a **bug** and a **problem**, the **incident lifecycle** (detect → declare → triage → mitigate → resolve → learn), why **restoring service comes before understanding**, and the vocabulary used in every later topic: **trigger, root cause, contributing factor, mitigation, blast radius, on-call, postmortem**.
>
> **You need:** nothing. This is the first topic.
>
> **Part 1 · Incident basics** · Next: [02 · Severity levels](02-severity-levels-and-impact.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** An **incident** is an **unplanned** event that **hurts users or the business** (or is about to), and needs people to **respond now**, in a coordinated way.

```
Something breaks → someone notices → someone declares "this is an incident"
→ people stop the damage → service is back → everyone learns from it
```

**Picture it.** A kitchen fire in a restaurant. The fire is the incident. The smoke alarm is the **alert**. The faulty wire that started it is the **root cause**. Using the extinguisher is the **mitigation**. Rebuilding the wiring is the **permanent fix**. The meeting next week where the staff ask "how did this happen, and how do we stop it happening again?" is the **postmortem**.

**The one thing to know.** During an incident your first job is **to stop the damage**, not to understand everything. Understanding comes later, when users are OK.

**Try it now:** think of the last time a website or app you use was down. Who was affected? For how long? What do you think the company did in the first 10 minutes?

**You've got it if you can:** explain the difference between an alert, an incident and a bug, and list the six lifecycle steps in order.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

An **incident** is any unplanned event that **disrupts or degrades a service** for its users, or **threatens to**, and needs an urgent response.

Three things make something an incident:

| Test | Question | Example |
|---|---|---|
| **Unplanned** | Did we choose to do this? | A planned maintenance window is **not** an incident. A maintenance that runs over and breaks checkout **is**. |
| **Impact (or risk of impact)** | Are users or the business hurt, or about to be? | "Checkout fails for 20% of users" (impact). "The database disk is 97% full and growing" (about to). |
| **Needs urgent action** | Can it wait until Monday? | A typo on the About page can wait. Login failing cannot. |

### Words you will see in every topic

| Word | Meaning in simple words |
|---|---|
| **Service** | A thing users depend on: a website, an API (Application Programming Interface), a payment system. |
| **Alert** | An automatic message from monitoring saying "a number crossed a line" (e.g. error rate > 5%). An alert is a **signal**, not an incident. Many alerts are false alarms. |
| **Page** | An alert loud enough to wake a human (phone call, push notification). "I got paged" = "my phone woke me up". |
| **On-call** | The engineer who is responsible for responding to pages during a time window (e.g. this week). |
| **Incident** | A real event with impact that people are now coordinating on. |
| **Declare** | Officially say "this is an incident", which starts the process (a channel, roles, updates). |
| **Trigger** | The event that **started** the failure (e.g. "a config file was pushed at 11:20"). |
| **Root cause** | The **underlying reason** the trigger could cause damage (e.g. "the software crashed on files bigger than a fixed limit, and nothing checked the file size"). |
| **Contributing factor** | Anything else that made it possible or worse (no staged rollout, a missing alert, a slow rollback). Real incidents almost always have **several**. |
| **Mitigation** | An action that **reduces or stops the impact**, even if the cause is still there (roll back, fail over, turn a feature off). |
| **Resolution** | The service is fully healthy again and stable. |
| **Permanent fix** | The change that stops this cause from hurting you again (fix the code, add a check, add a guardrail). |
| **Blast radius** | How much was affected: one server, one region, every customer in the world. |
| **Postmortem** | A written review after the incident: what happened, why, and what we will change. Also called an **incident review**, **post-incident review (PIR)** or **RCA (Root Cause Analysis)**. |
| **Blameless** | The postmortem looks for **what in the system** allowed the mistake, not **who** to punish (topic 11). |

### Alert vs incident vs bug vs problem

| Term | What it is | Example | Who handles it, and how fast |
|---|---|---|---|
| **Alert** | An automatic signal | "CPU > 90% on web-3" | On-call looks at it; may be nothing |
| **Incident** | Real impact, urgent, coordinated | "Web-3's CPU is 100% and 30% of requests are timing out" | On-call now, maybe a whole team |
| **Bug** | A defect in code | "The discount code is applied twice" | The team, in normal work, by priority |
| **Problem** (ITIL term) | The **unknown underlying cause** of one or more incidents | "Why does web-3 run out of CPU every Friday?" | Investigated after, through problem management |

**ITIL** (Information Technology Infrastructure Library) is a widely used set of IT service management practices. It separates **incident management** (restore service fast) from **problem management** (find and remove the cause). The SRE (Site Reliability Engineering) world usually does both in the postmortem, but the separation is a useful idea: **fixing now** and **understanding deeply** are different jobs, done at different speeds.

---

## Why it exists

Without a defined incident process, outages turn into chaos:

- Ten people debug the same thing and nobody checks the other suspect.
- Two engineers make conflicting changes at the same moment, and now nobody knows which one helped or hurt.
- Customers hear nothing for an hour, then flood the support team.
- Nobody writes down what happened, so the **same failure happens again** next month.

An incident process is a small set of **habits agreed in advance**, so that on the worst day people do not have to invent how to work together. It is the same reason airlines use checklists and hospitals run resuscitation drills.

---

## Analogy

**A hospital emergency room.**

| ER | Production |
|---|---|
| A patient arrives | An alert fires or a user reports a problem |
| **Triage** nurse decides how urgent it is | You set the **severity** (topic 02) |
| Stabilise the patient first (stop bleeding, oxygen) | **Mitigate** first (roll back, fail over) |
| Diagnose the illness later with tests | Find the **root cause** after service is restored |
| One doctor leads the resuscitation and gives orders | The **Incident Commander** leads (topic 05) |
| Notes are taken on everything | The **Scribe** keeps a timeline |
| Case review after a death or near miss | **Postmortem** (topic 11) |

A doctor does not order a full genetic test while the patient is bleeding. You should not read source code for an hour while checkout is down and a rollback button is available.

---

## How it works: the incident lifecycle

```
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────┐   ┌──────────┐
 │ 1 DETECT │──►│2 DECLARE │──►│ 3 TRIAGE │──►│4 MITIGATE │──►│5 RESOLVE │──►│ 6 LEARN  │
 └──────────┘   └──────────┘   └──────────┘   └───────────┘   └──────────┘   └──────────┘
  alert, user     open channel,   who is hurt?    stop the        stable,        postmortem,
  report,         page people,    how badly?      bleeding:       monitored,     action items,
  synthetic       set severity,   since when?     rollback,       incident       follow-up
  check           assign IC       what changed?   failover...     closed         tracked to done
                                         ▲                │
                                         └── didn't work ─┘  (try the next mitigation)
```

**Step by step:**

1. **Detect.** Something notices: an alert, a synthetic check (a robot that pretends to be a user every minute), an engineer, or (worst case) a customer on social media. *How fast* you detect is measured as **MTTD**, Mean Time To Detect (topic 10).
2. **Declare.** Someone says "this is an incident". A chat channel is opened, a severity is set, and an **Incident Commander (IC)** is named. Declaring is cheap; you can always downgrade.
3. **Triage.** Quick questions, in this order: *What do users see? How many? Since when? What changed just before?* (topic 07).
4. **Mitigate.** Apply the **fastest safe action** that reduces impact. Very often: **roll back the last change**. If it doesn't help, try the next one. One change at a time.
5. **Resolve.** Metrics are back to normal and stay there. The IC closes the incident (or downgrades it while someone keeps watching).
6. **Learn.** A blameless postmortem within a few days: timeline, root cause and contributing factors, and **action items with owners and due dates**.

> ⚠️ **"Resolved" is not "fixed".** Rolling back stops the impact (resolved), but the bad code still exists in the main branch. If someone deploys it again tomorrow, the incident comes back. The permanent fix is an action item.

### Trigger vs root cause: a first example

Imagine: "The site went down after we deployed version 2.4."

- **Trigger:** deploying version 2.4.
- **Root cause:** version 2.4 opened a new database connection on every request and never closed it, so the database hit its connection limit.
- **Contributing factors:** no load test with production traffic; no alert on database connection count; the deploy went to 100% of servers at once instead of a small canary first.

A junior answer stops at the trigger ("a bad deploy"). A senior answer explains the **root cause** and the **contributing factors**, and why the safeguards did not catch it. Every outage in Part 3 is written this way.

---

## Failure modes (of handling incidents)

| What goes wrong | What it looks like | How to avoid it |
|---|---|---|
| **Nobody declares** | Everyone thinks someone else is on it; impact grows quietly | Rule: if in doubt, declare. Anyone may declare. |
| **Debugging before mitigating** | One hour of log reading while a rollback was available | "Mitigate first" norm (topic 07) |
| **Too many cooks** | 30 people in a call, all typing commands | One IC, one Ops lead; others listen or leave |
| **Silence** | Customers and support hear nothing | A Comms role and a fixed update cadence (topic 06) |
| **No learning** | The same outage returns in a month | Postmortem + tracked action items (topic 11) |
| **Blame** | People hide what they did; the timeline is wrong | Blameless culture (topic 11) |

---

## Where you'll use it

- **Every on-call shift:** deciding "is this alert an incident?" is your first judgement call.
- **Your other notes:** Docker break-fix labs ([Docker 52](../../Docker/notes/52-production-scenarios.md)), Kubernetes break-fix ([K8s 84](../../CKA-Certification-Course-2025/notes/84-production-break-fix-scenarios.md)), Terraform incidents ([Terraform 66](../../Terraform/notes/66-production-scenarios.md)), Linux scenarios ([Linux 110](../../linux-mastery-rhel/notes/110-production-scenarios.md)) are all incidents seen from one tool. This path shows the **whole** picture.
- **System design:** [SD 30 · Incident management](../../System-Design/notes/30-incident-management.md) is the design-level summary of Part 1.
- **Interviews:** "Tell me about an incident you handled" is asked in almost every senior DevOps/SRE interview (topic 91).

---

## Quick recap

- Incident = **unplanned + impact (or imminent impact) + urgent**.
- Alert = signal. Incident = coordinated response. Bug = code defect. Problem = unknown underlying cause.
- Lifecycle: **Detect → Declare → Triage → Mitigate → Resolve → Learn.**
- **Mitigate first**, understand later. Resolved is not the same as fixed.
- Separate the **trigger** from the **root cause** and the **contributing factors**.

### Interview questions

1. **Q: What is the difference between an alert and an incident?**
   A: An alert is an automatic signal that a metric crossed a threshold; it may be noise. An incident is a confirmed event with user or business impact (or imminent impact) that people are actively coordinating to resolve. One incident often produces many alerts, and many alerts never become incidents.

2. **Q: What's the difference between mitigation and a fix?**
   A: Mitigation stops or reduces the impact now, without necessarily removing the cause (roll back, fail over, disable a feature). The fix removes the cause so it can't recur (correct the code, add validation, add a guardrail). You mitigate during the incident and fix afterwards, through postmortem action items.

3. **Q: Why don't you look for the root cause first?**
   A: Because users are hurting while you look. A rollback or failover often takes minutes and is reversible; root-causing can take hours. Restoring service first also lowers the pressure, so the investigation afterwards is calmer and more accurate. The exception is when you can't mitigate without knowing more, and even then you look for the fastest safe action.

4. **Q: What's the difference between a trigger and a root cause?**
   A: The trigger is the event that started the failure (a deploy, a config push, a traffic spike). The root cause is the underlying weakness that let that trigger cause damage (no input validation, no limit, a hidden dependency). Fixing only the trigger ("don't push that config again") leaves the weakness for the next trigger.

5. **Q: Is a planned maintenance an incident?**
   A: Not by itself, because it is planned and communicated. It becomes an incident if it causes unplanned impact, for example it runs over its window, or breaks something it wasn't supposed to touch.
