# 03 · SLOs and Error Budgets for On-Call

> **Level:** ★ foundation
>
> **You will learn:** the four words that decide what "broken" means: **SLI** (Service Level Indicator), **SLO** (Service Level Objective), **SLA** (Service Level Agreement) and **error budget**, how to turn "99.9%" into **minutes of allowed downtime**, what **burn rate** is and why it is the best thing to page on, and how SLOs connect to **severity** and **postmortems**.
>
> **You need:** topics 01–02.
>
> **Part 1 · Incident basics** · Previous: [02 · Severity levels](02-severity-levels-and-impact.md) · Next: [04 · On-call and alerting](04-on-call-and-alerting.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** You can't respond well to "the site is broken" unless you have agreed what "working" means, in numbers.

```
SLI  = what we measure         "% of HTTP requests that succeed in < 300 ms"
SLO  = our target for the SLI  "99.9% over 30 days"
SLA  = a promise with a penalty "99.5%, or customers get a refund"
Error budget = 100% − SLO      "0.1% of requests may fail" ≈ 43 minutes / 30 days
```

**Picture it.** A monthly mobile data plan. The SLO is your plan size. Each failed request "spends" a little budget. If you burn the whole month's budget in two days, something is very wrong **now** (page). If you're slowly using a bit more than usual, look at it on Monday (ticket).

**The one thing to know.** Page on **how fast the error budget is burning** (burn rate), not on raw CPU or memory. Users feel errors and latency, not CPU.

**Try it now:** compute the monthly downtime allowed by 99.9% and 99.99% (answer below).

**You've got it if you can:** explain SLI/SLO/SLA/error budget with one example, and say why a burn-rate alert beats a CPU alert.

> 📖 **That's the core.** Everything below is the **full reference**. The full design-level treatment is [SD 28 · SLI, SLO, SLA and error budgets](../../System-Design/notes/28-sli-slo-sla-error-budgets.md).

---

## What it is

| Term | Full form | Simple meaning | Example |
|---|---|---|---|
| **SLI** | Service Level **Indicator** | A **measurement** of how the service looks to users | Successful requests ÷ total requests |
| **SLO** | Service Level **Objective** | Our internal **target** for that measurement | 99.9% of requests succeed, over a rolling 30 days |
| **SLA** | Service Level **Agreement** | A **contract** with customers, with penalties (credits, refunds) | 99.5% monthly uptime, else 10% credit |
| **Error budget** | — | How much failure the SLO **allows** | 0.1% of requests in 30 days |
| **Burn rate** | — | How fast you are using the budget, compared with "evenly over the month" | Burn rate 1 = budget lasts exactly 30 days; burn rate 14.4 = budget gone in ~2 days |

**Rule:** SLA < SLO. Your internal target must be stricter than your contract, so you notice trouble before you owe money.

### Nines → downtime

| Availability | Downtime per 30 days | Downtime per year |
|---|---|---|
| 99% | 7.2 hours | ~3.65 days |
| 99.9% ("three nines") | ~43.2 minutes | ~8.8 hours |
| 99.95% | ~21.6 minutes | ~4.4 hours |
| 99.99% ("four nines") | ~4.3 minutes | ~53 minutes |

Math: 30 days = 43,200 minutes. 0.1% of 43,200 = 43.2 minutes.

**Good SLIs** (from users' point of view):
- **Availability:** % of requests that don't return 5xx errors.
- **Latency:** % of requests faster than a threshold (e.g. 300 ms). Use percentiles: **p99** = 99% of requests were at least this fast.
- **Correctness / freshness:** % of jobs that produced correct output; data no older than N minutes.

---

## Why it exists

Without SLOs, "is this an incident?" is a feeling. With SLOs:

- **"Broken" has a number.** 2% errors for 5 minutes may be fine for one service and a disaster for another. The SLO tells you which.
- **Paging is fair.** You get woken up only when users are hurt fast enough to matter.
- **Trade-offs are explicit.** If the error budget is used up, the team slows feature releases and works on reliability (an **error budget policy**).
- **Postmortems have numbers.** "This incident burned 60% of the monthly error budget" is clear to everyone, including executives.

---

## Analogy

**A monthly phone data plan (10 GB).** The SLI is how much data you used. The SLO is "stay under 10 GB". The error budget is the 10 GB itself. The burn rate is how fast you're using it. Watching one 4K video at 2 AM burns fast. You'd want a notification **immediately** ("at this speed you'll run out in 3 hours"). Using slightly more than usual every day only needs a weekly email.

---

## How it works

### Burn rate in one picture

```
Error budget (30 days)
100% ┤█████████████████████████████████
     │                    ╲  normal burn (rate 1): lasts 30 days
     │      ╲              ╲
     │       ╲ fast burn    ╲
     │        ╲ (rate 14.4)  ╲
  0% ┤─────────╲──────────────╲──────────►
     0         2 days           30 days
```

**Burn rate = (observed error rate) ÷ (error rate allowed by the SLO).**

Example: SLO 99.9% → allowed error rate 0.1%. Right now 1.44% of requests fail. Burn rate = 1.44 ÷ 0.1 = **14.4**. At that speed the whole 30-day budget is gone in 30 ÷ 14.4 ≈ **2.1 days**.

### Multi-window burn-rate alerts

The Google SRE Workbook recommends alerting on burn rate over **two windows** at once (a long one to be sure it is real, a short one to be sure it is still happening). A commonly used starting set:

| Severity of alert | Burn rate | Long window | Short window | Budget used if it continues for the long window |
|---|---|---|---|---|
| **Page** | 14.4 | 1 hour | 5 minutes | 2% |
| **Page** | 6 | 6 hours | 30 minutes | 5% |
| **Ticket** | 1 | 3 days | 6 hours | 10% |

Why two windows: the long window avoids paging on a 30-second blip; the short window makes the alert **stop** quickly once you fix things.

Example as a Prometheus-style rule (illustrative):

```yaml
- alert: CheckoutErrorBudgetFastBurn
  expr: |
    (sum(rate(http_requests_total{job="checkout",code=~"5.."}[1h]))
      / sum(rate(http_requests_total{job="checkout"}[1h]))) > (14.4 * 0.001)
    and
    (sum(rate(http_requests_total{job="checkout",code=~"5.."}[5m]))
      / sum(rate(http_requests_total{job="checkout"}[5m]))) > (14.4 * 0.001)
  labels:
    severity: page
  annotations:
    summary: "Checkout is burning its 30-day error budget 14x too fast"
    runbook: "https://runbooks.example.internal/checkout/high-errors"
```

What each part tells you:
- `rate(...[1h])` = average per-second rate over the last hour. The ratio is the **error rate**.
- `> (14.4 * 0.001)` = "error rate is more than 14.4 × the allowed 0.1%".
- The `and` with the 5-minute window = "and it is **still** happening".
- `runbook:` = a link the on-call opens first (topic 09).

### SLOs during an incident

- **Triage:** "How much budget are we burning?" turns a vague feeling into a severity.
- **Done?** The incident is resolved when the SLI is back inside the SLO and stays there.
- **Postmortem:** record budget consumed. Many teams require a postmortem for any incident that used more than ~20% of a month's budget (this threshold is a common choice, not a standard).

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **SLI measured in the wrong place** | Server metrics say 100% OK, users see errors (the load balancer or CDN failed in front of the servers) | Measure at the edge / load balancer, plus synthetic checks |
| **SLO too strict** | Pages every day; team ignores them | Set the SLO to what users actually need, not 100% |
| **SLO too loose** | Users complain while dashboards are green | Tighten; add latency SLIs |
| **Averages instead of percentiles** | Average latency 80 ms, but 5% of users wait 10 seconds | Use p95/p99 or "% under threshold" |
| **No error budget policy** | Budget gone, team keeps shipping risky changes | Agree in advance what happens when budget runs out |

---

## Where you'll use it

- **Alerting design:** topic 04 and [SD 29](../../System-Design/notes/29-alerting-and-on-call.md).
- **Kubernetes:** Prometheus + Alertmanager from [K8s 80 · Production observability](../../CKA-Certification-Course-2025/notes/80-production-observability.md) is where these rules usually live.
- **Postmortems:** "impact" sections of every Part 3 outage use these ideas.

---

## Quick recap

- SLI = measurement; SLO = target; SLA = contract; error budget = 100% − SLO.
- 99.9% ≈ 43 minutes per 30 days; 99.99% ≈ 4.3 minutes.
- Burn rate = observed error rate ÷ allowed error rate.
- Page on fast burn (two windows), ticket on slow burn.
- Measure SLIs where users are (edge, synthetic checks).

### Interview questions

1. **Q: Explain SLI, SLO and SLA with an example.**
   A: For an API, the SLI might be the percentage of requests that succeed in under 300 ms. The SLO is our internal target, say 99.9% over 30 days. The SLA is the contract with customers, say 99.5%, with service credits if we miss it. The SLA is looser than the SLO so we have warning before we owe money.

2. **Q: Why alert on burn rate instead of error rate or CPU?**
   A: CPU is a cause, not a symptom; high CPU can be harmless and low CPU can hide a broken service. A raw error-rate threshold doesn't know the SLO. Burn rate tells you how fast users are being hurt relative to what you promised, so a page means "at this speed we break our objective soon". Multi-window burn-rate alerts are both fast for big outages and quiet for blips.

3. **Q: Your service is at 99.95% this month and the SLO is 99.9%. A risky deploy is planned. What do you do?**
   A: There's budget left (we've used about half), so the deploy can go, but with normal safety: canary, automated rollback, deploying when the team is around. If the budget were exhausted, the error budget policy would usually freeze risky changes until reliability work is done.

4. **Q: Where should you measure availability?**
   A: As close to the user as possible: at the load balancer or CDN logs, plus synthetic checks from outside. Measuring only on the servers misses failures in front of them, such as DNS, certificates or the load balancer itself.
