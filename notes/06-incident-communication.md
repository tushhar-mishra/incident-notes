# 06 · Incident Communication

> **Level:** ★ foundation
>
> **You will learn:** **who** needs to hear about an incident (responders, support, leadership, customers), **where** (incident channel, status page, email), **how often** (update cadence), ready-to-use **templates** for internal updates and status-page posts, what **not** to say publicly, and what to do when **your communication tools are down too**.
>
> **You need:** topics 01–05.
>
> **Part 1 · Incident basics** · Previous: [05 · Incident roles](05-incident-roles-and-commander.md) · Next: [07 · The first 15 minutes](07-the-first-15-minutes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** During an outage, **silence is the worst message**. People assume you don't know or don't care. Communicate on a **fixed schedule**, even if the update is "still investigating".

```
Responders     → incident channel (#inc-2026-11-20-checkout) + call   (continuous)
Company        → short internal update (every 15–30 min for SEV1)
Support team   → what to tell customers, workarounds                  (each update)
Leadership     → impact, ETA if known, business risk                  (SEV1/SEV2)
Customers      → status page in plain language                        (SEV1/SEV2, every 30–60 min)
```

**Picture it.** A delayed flight. The worst airline says nothing for two hours. The good airline announces every 20 minutes: "Still waiting for a part. Next update at 3:20." Passengers are still unhappy, but they trust the airline.

**The one thing to know.** Every update ends with **"next update at HH:MM"**, and you keep that promise.

**Try it now:** write a status-page message for "logins failing for some users in Europe" using the template below.

**You've got it if you can:** write an internal update and a public status update for the same incident, and explain why they're different.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Audience | Channel | What they need | How often |
|---|---|---|---|
| **Responders** | Incident chat channel + video/voice bridge | Facts, hypotheses, actions, decisions | Continuous |
| **Rest of engineering** | Announcement channel (#incidents) | "There's an incident, don't deploy to X, here's the channel" | At start, major changes, end |
| **Support / customer success** | Their channel or ticket macro | What customers see, workaround, what to say | Every update |
| **Leadership** | Short summary (chat/email) | Impact, business risk, ETA (if any), what help is needed | SEV1: every 30 min; SEV2: hourly |
| **Customers** | Status page, in-app banner, email for big accounts | What's affected, whether there's a workaround, when the next update comes | SEV1/SEV2: every 30–60 min |
| **Regulators / legal** | Via legal team | For data breaches or regulated services | As required by law/contract |

---

## Why it exists

- **Support gets flooded** if customers learn nothing. A clear status page cuts tickets.
- **Engineers get interrupted** by "what's going on?" questions if nobody broadcasts updates.
- **Trust** is lost more by silence than by the outage itself.
- **Contracts and laws** may require notification within set times (e.g. data-breach rules).

---

## Analogy

**A delayed flight** (Core). The gate agent (Comms) talks to passengers; the pilot and engineers (Ops) fix the plane. Passengers don't go into the cockpit to ask questions, because the gate agent tells them what they need on a schedule.

---

## How it works

### Templates

**Internal update (responders + company):**

```
[SEV1] Checkout errors: update #3 at 14:30 UTC
Status:   Mitigating (Investigating / Identified / Mitigating / Monitoring / Resolved)
Impact:   ~25% of checkout attempts failing since 13:58 UTC. All regions.
Cause:    Suspected: payments-svc v2.14 (deployed 13:58). Not confirmed.
Actions:  Rolling back to v2.13 (ETA 14:40). Payment provider status also degraded.
Asks:     Nobody deploy payments-svc. Support: use macro "checkout-delay".
IC: Priya · Ops lead: Sam · Comms: Lee
Next update: 15:00 UTC
```

**Public status page:**

```
Investigating: Some customers may be unable to complete checkout
We're aware that some customers are seeing errors when completing a purchase.
Our team is actively working on a fix. Items in your cart are saved.
Next update in 30 minutes.

Update: We've identified the cause and are applying a fix. Checkout is
recovering for most customers. Next update by 15:00 UTC.

Resolved: Checkout is working normally. Between 13:58 and 14:52 UTC some
customers were unable to complete purchases. We're sorry for the trouble
and will share what we're doing to prevent this.
```

### Rules for public messages

| Do | Don't |
|---|---|
| Plain language about what users see | Internal names ("payments-svc pod OOMKilled in eks-prod-2") |
| Say what still works ("your cart is saved") | Blame a vendor or a person |
| Give the next update time | Promise a fix time you can't keep |
| Acknowledge quickly, even with little info | Wait until you know the cause |
| Get legal/security review for breaches | Speculate about security or data loss |

### Status stages

`Investigating → Identified → Monitoring → Resolved`. Most status page tools (Atlassian Statuspage, Instatus, etc.) use these words. Use them in internal updates too, so everyone speaks the same language.

### When your own tools are down

Big outages often take down the tools you'd use to talk about them:

- **Facebook 2021** (topic 59): internal tools and badge systems depended on the same network that was down.
- **Slack** is often the incident channel. If Slack is down, or your SSO (Single Sign-On) is down, can you still talk?
- Your **status page** may be hosted on the same cloud or CDN that's failing.

Prepare a **backup channel**: a phone bridge number printed in the runbook, a second chat tool, an SMS group, and a status page hosted **independently** of your main infrastructure.

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Silence** | No update for an hour; support flooded | Comms role + "next update at" rule |
| **Over-promising** | "Fixed in 10 minutes" → 3 hours later | Say what you're doing, not when it'll be done, unless sure |
| **Status page lies** | All green while users are down (often manually updated) | Comms lead updates it at declaration; consider automating "investigating" from SLO alerts |
| **Too much detail publicly** | Leaks internal info, confuses customers | Templates and review |
| **Comms tools share fate** | Can't reach the team because chat/SSO/status page are down | Independent backup channels, practised |

---

## Where you'll use it

- Every SEV1/SEV2 in this path; the outages in Part 3 each mention how the company communicated.
- **CI/CD:** a "deploy freeze" announcement during an incident ([SD 33](../../System-Design/notes/33-releases-promotion-gitops.md)).
- **Interviews:** "How did you keep stakeholders informed?" is part of the STAR answer in topic 91.

---

## Quick recap

- Communicate on a schedule; always give the next update time.
- Different audiences need different messages: responders, company, support, leadership, customers.
- Public messages: plain language, what still works, no blame, no internal names.
- Status stages: Investigating → Identified → Monitoring → Resolved.
- Have backup channels independent of your main infrastructure.

### Interview questions

1. **Q: What do you put in an incident update?**
   A: Status stage, current impact (who, how much, since when), what we think the cause is (and whether it's confirmed), current actions with ETAs if known, any asks (e.g. deploy freeze), the roles, and the time of the next update.

2. **Q: Should you post on the status page before you know the cause?**
   A: Yes. Customers already see the problem; acknowledging it quickly builds trust and reduces support load. "We're investigating errors with checkout; next update in 30 minutes" is enough.

3. **Q: Your status page is hosted on the same cloud provider that's having the outage. What's the problem?**
   A: The status page can go down together with the service, exactly when customers need it. Host it independently (a different provider, or a SaaS status page), and make sure the people who update it can log in without depending on the failing systems.
