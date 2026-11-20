# 02 · Severity Levels and Impact

> **Level:** ★ foundation
>
> **You will learn:** how to **measure impact** (who, how many, how badly, how long), how companies define **severity levels (SEV1–SEV4)**, what each level **triggers** (who gets paged, how often you update, who must be told), **when to declare**, how to **upgrade and downgrade** severity, and the special cases: **security incidents, data loss, and near misses**.
>
> **You need:** topic 01.
>
> **Part 1 · Incident basics** · Previous: [01 · What is an incident](01-what-is-an-incident.md) · Next: [03 · SLOs and error budgets](03-slos-and-error-budgets-for-on-call.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Severity** (SEV) is a label for **how bad** an incident is. The label decides **how big the response is**: who gets woken up, how often you update, whether executives and customers are told.

```
SEV1  Critical  everyone affected / data loss / security breach  → all hands, now
SEV2  Major     big part of users or a key feature broken       → on-call + IC, now
SEV3  Minor     small impact, a workaround exists                → on-call, in hours
SEV4  Low       no user impact yet (near miss, lost redundancy)  → ticket
```

Some companies number the other way (P1/P2 for "priority"), or use SEV0 for "worse than SEV1". The idea is always the same.

**Picture it.** Hospital triage colours: red (now), yellow (soon), green (can wait). The colour isn't the diagnosis; it decides **how fast and how many** people come.

**The one thing to know.** **When in doubt, declare higher.** Downgrading a false SEV1 costs a few people 10 minutes. Treating a real SEV1 like a SEV3 costs hours of outage.

**Try it now:** for a service you know, write one example of each SEV level.

**You've got it if you can:** given a symptom ("10% of logins fail in Europe"), pick a severity and explain why.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

### Measuring impact: the five questions

Severity comes from **impact**. Ask:

| Question | Example answers |
|---|---|
| **Who** is affected? | All users / one region / one customer / internal staff only |
| **How many**? | 100% of requests / 5% / one big customer |
| **How badly**? | Can't use the product at all / slow / one feature broken / cosmetic |
| **How long** so far, and is it growing? | 2 minutes and stable / 40 minutes and getting worse |
| **What is at risk**? | Money (checkout), data (corruption, loss), security (leak), legal (SLA, regulator) |

### A typical severity table

| SEV | Definition | Examples | Response |
|---|---|---|---|
| **SEV1** | Critical. Core product down or unusable for many users; data loss or corruption; confirmed security breach; legal/safety risk | Checkout down for everyone; database deleted; customer data exposed | Page on-call **and** IC immediately; exec notified; status page within ~15 min; updates every 15–30 min; all needed teams join |
| **SEV2** | Major. Significant degradation or a key feature broken for a large group | Login failing for 20% of users; one region down; search returning errors | Page on-call + IC; status page; updates every 30 min |
| **SEV3** | Minor. Limited impact or a workaround exists | Reports slow; one internal tool down; a non-critical API has elevated errors | On-call handles during working hours; internal updates |
| **SEV4** | Low / near miss. No user impact yet | A replica died but traffic is fine; a backup failed once; a certificate expires in 10 days | Ticket, fix in normal work |

These are **examples**. Every company writes its own table. What matters is that it is **written down before** the incident, so nobody argues about it at 3 AM.

### Special cases

- **Security incidents** (a breach, a leaked key in use, malware) usually follow a **separate process** with the security team, legal, and sometimes regulators. They are SEV1 by default until proven otherwise. Evidence must be preserved (don't wipe the server you are investigating).
- **Data loss or corruption** is SEV1 even if the site looks "up". A site that is up but writing wrong data is worse than a site that is down, because the damage keeps spreading.
- **Near misses** ("we were lucky") deserve a SEV4 and often a short postmortem. They are free lessons.

---

## Why it exists

People respond to an incident in proportion to how scary it **feels**, and feelings are unreliable at 3 AM. A written severity table:

- **Removes debate.** "Is this worth waking up the database team?" is answered by the table.
- **Sets expectations.** Everyone knows SEV1 means updates every 15–30 minutes.
- **Protects people.** A SEV3 doesn't wake up five people. Over-paging burns people out (topic 04).
- **Drives follow-up.** Many companies require a postmortem for every SEV1 and SEV2.

---

## Analogy

**Fire alarms in a big building.** A smoke detector in one office calls building security (SEV3). Fire in a corridor triggers the floor alarm and the fire brigade (SEV2). Fire spreading to several floors means full evacuation and every engine in the city (SEV1). Nobody decides the response from scratch each time. The building has a plan, and the **level** chooses which part of the plan to run.

---

## How it works

### When to declare

Declare an incident when **any** of these is true:

- Users are impacted and you can't fix it in a few minutes on your own.
- You need help from another team.
- You're not sure how bad it is, but it **could** be bad.
- A customer-facing SLA (Service Level Agreement, a contractual promise, topic 03) might be breached.
- Data or security is at risk.

> **3 AM rule:** if you have been looking at something for 15 minutes and it is getting worse, declare. Declaring is cheap.

### Upgrading and downgrading

Severity is **not fixed**. The IC changes it as facts arrive:

```
14:02  SEV3 declared: "elevated errors on search"
14:10  Errors spreading to checkout → upgrade to SEV2
14:15  Checkout fully down for all users → upgrade to SEV1, exec paged, status page
14:40  Rollback done, checkout recovered, search still slow → downgrade to SEV3
15:30  All metrics normal for 30 min → resolved
```

The **highest** severity reached is usually the one recorded for the postmortem.

### Severity vs priority

- **Severity** = how bad the **impact** is.
- **Priority** = how soon we **work on** it.

They usually match. They can differ: a small bug in the payment receipt might be low severity but high priority before a big sale day.

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Under-declaring** | A SEV1 handled by one tired person for an hour | "When in doubt, declare higher"; anyone may declare |
| **Over-declaring everything** | Every blip is a SEV1; people stop taking SEV1 seriously | Clear definitions with examples; review in retros |
| **Severity by loudness** | The VP's complaint becomes SEV1, a silent data corruption stays SEV3 | Use the five impact questions, not who is shouting |
| **No security path** | Engineers "clean up" a hacked server and destroy evidence | Separate security incident process, known in advance |
| **Never downgrading** | Incident stays SEV1 for 8 hours, everyone exhausted | IC downgrades when impact ends; hand monitoring to one person |

---

## Where you'll use it

- **Alert routing:** your paging tool (PagerDuty, Opsgenie, Grafana OnCall, etc.) uses severity to decide whom to page and how loudly (topic 04).
- **Status pages:** SEV1/SEV2 usually require a public update (topic 06).
- **SLOs:** severity can be tied to how fast you're burning error budget (topic 03).
- **System Design:** the severity table in [SD 30](../../System-Design/notes/30-incident-management.md) is the same idea at design level.

---

## Quick recap

- Severity = how bad the impact is → decides the size of the response.
- Measure impact with five questions: who, how many, how badly, how long, what's at risk.
- SEV1 critical · SEV2 major · SEV3 minor · SEV4 near miss.
- Data loss and security are SEV1 by default.
- Declare early, declare high, change severity as facts arrive.

### Interview questions

1. **Q: How do you decide the severity of an incident?**
   A: By user and business impact, not by how the alert looks. I ask: who is affected, how many, how badly, for how long and is it growing, and what's at risk (money, data, security, contracts). Then I map that to the company's written table. If I'm unsure, I pick the higher level and downgrade once I have facts.

2. **Q: The site looks fine but a bug is writing wrong prices into the database. What severity?**
   A: SEV1. Silent data corruption keeps spreading and can be harder to recover from than an outage. I'd stop the writes first (feature flag off, roll back, or put the writer in read-only mode), then assess how much data is affected and plan a repair from backups or logs.

3. **Q: What's the difference between severity and priority?**
   A: Severity measures the impact; priority is the order in which we work on things. They usually line up, but a low-severity issue can be high priority because of timing (for example right before a launch).

4. **Q: Why would you declare an incident that turns out to be minor?**
   A: Because the cost is a few minutes of a few people's time, while the cost of under-reacting to a real SEV1 is a long outage. Teams that punish false alarms get late declarations.
