# 39 · Third-Party and Cloud-Region Outages

> **Level:** ★★ intermediate
>
> **You will learn:** what to do when the problem is **not yours**: a cloud region degraded, a CDN or DNS provider down, a payment/email/auth vendor failing; how to **confirm** it quickly (and why provider status pages lag), **hidden dependencies** (your status page, SSO, CI, container registry, monitoring all on the same provider), **control plane vs data plane**, why **us-east-1** shows up so often, **static stability**, and realistic mitigation (failover, degradation, multi-region, multi-provider — and their costs).
>
> **You need:** topics 01–38. Helpful: [SD 42 · Multi-region architecture](../../System-Design/notes/42-multi-region-architecture.md), [SD 41 · Backups and DR](../../System-Design/notes/41-backups-and-disaster-recovery.md).
>
> **Part 2 · Recurring failures · F. Overload and cascades** · Previous: [38 · Cascading failures](38-cascading-failures.md) · Next: [40 · rm -rf and the wrong environment](40-rm-rf-and-wrong-environment.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Your service depends on other companies' services. When one fails, you can't fix it, but you **can** decide how much it hurts you: **detect it fast**, **communicate**, **degrade gracefully**, or **fail over** if you planned for it.

```
Your app ──► Cloud region (compute, DB, queues, IAM, DNS)
         ──► CDN / DNS provider
         ──► SaaS: payments, email, auth (SSO), feature flags, monitoring, CI, registry, chat, status page
Any one fails → how much of YOUR product still works?
```

**Picture it.** A restaurant whose fish supplier didn't deliver. You can't fix the supplier's truck. You **can** take fish off the menu, put up a sign, and use a backup supplier if you arranged one in advance.

**The one thing to know.** **Know your dependencies before the incident**, including the hidden ones (your SSO, CI, monitoring and status page). The worst moment to discover that your failover runbook lives on a wiki hosted in the failed region is during the failover.

**Try it now:** list every external service your product needs to take an order. Mark which ones you could live without for an hour.

**You've got it if you can:** confirm a provider outage in minutes and describe the degradation plan for your top three dependencies.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: [62 · AWS us-east-1, October 2025](62-aws-dynamodb-dns-2025.md), [68 · AWS us-east-1, December 2021](68-aws-us-east-1-2021.md), [55 · Cloudflare November 2025](55-cloudflare-nov-2025.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Your errors | Timeouts/5xx only on calls to one provider or one region's services |
| Breadth | Many unrelated companies having trouble at the same time (social media, outage trackers) |
| Cloud | AWS API errors (`InternalError`, `ServiceUnavailable`), instance launches failing, IAM/STS calls slow |
| Your tools | You can't log into the console (SSO depends on the same provider), CI can't deploy, monitoring has gaps |
| Provider status page | Often **green** for the first part of the outage (status updates usually trail detection) |

---

## Analogy

The **fish supplier** (Core).

---

## How it happens

### Control plane vs data plane

- **Data plane:** the part that serves your running traffic (EC2 instances running, S3 GET/PUT, load balancers forwarding).
- **Control plane:** the part that **changes** things (launch instances, create LBs, update DNS, IAM changes).

Control planes are more complex and fail more often; data planes are built to keep running without them. **Static stability** means designing so you don't **need** the control plane during a failure: e.g. having enough instances already running in each AZ instead of planning to launch new ones when an AZ fails.

### Why us-east-1 appears so often

It's AWS's oldest and largest region, and some **global** AWS features have historically had control-plane dependencies there (AWS has been reducing these). Many companies also run only there. When it has problems, a large part of the internet notices (topics 62 and 68).

### Hidden dependencies

| You think you depend on… | You also depend on… |
|---|---|
| Your app in region X | Your container registry, secrets manager, KMS, IAM, DNS — maybe in the same region |
| Your monitoring | Monitoring SaaS hosted on the same cloud/region |
| Your incident process | Slack, Zoom, PagerDuty, SSO (Okta/Entra ID), the wiki with the runbooks |
| Your status page | Hosted behind the same CDN/cloud |
| Your CI/CD | GitHub/GitLab, runners, artifact storage |

---

## How to find it

**First:** is it us or them? Check **our** metrics broken down by dependency.

```
sum by (dependency) (rate(client_requests_total{status=~"5..|timeout"}[5m]))
```
- Errors only for `payments-provider` → their side (or our path to them).

```bash
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' https://api.provider.com/health
dig api.provider.com +short           # DNS to them OK?
```

**Second:** confirm externally: provider status page **and** independent signals (outage trackers, other customers, your account's health dashboard, which can show issues before the public page).

```bash
aws health describe-events --region us-east-1 \
  --filter eventStatusCodes=open --query 'events[].[service,eventTypeCode,region,startTime]' --output table
```

**Third:** map the blast radius on **your** side: which features, which regions, which internal tools are affected?

---

## How to fix it

### Immediate

- **Declare** and **communicate** (topic 06): "Our payment provider is experiencing an outage; card payments may fail."
- **Degrade:** turn off features that need the failed dependency; queue work for later (e.g. send emails when the provider recovers); show cached data.
- **Fail over** if prepared: secondary provider (payments, DNS, CDN, email), another region (warm standby / active-active).
- **Avoid making it worse:** don't retry aggressively (topic 36); **don't deploy** or make big changes in an impaired region (control plane may be broken, and you may lose capacity you can't get back).
- **Wait well:** keep the incident open, communicate on cadence, prepare for the recovery wave (backlogs, retries, cold caches — topic 37).

### Permanent

- **Dependency inventory** with criticality, owner, fallback.
- **Timeouts and circuit breakers** per dependency; graceful degradation designed per feature.
- **Static stability:** pre-provisioned capacity across AZs; cached credentials/config.
- **Multi-region / multi-provider** only where the business case justifies the cost and complexity — and **test failover regularly**, or it won't work.
- Incident tools independent of production: status page, chat backup, runbooks available offline.

---

## How to verify the fix

- Provider recovered (their status + **your** metrics for calls to them back to normal).
- Backlogs drained, queued work processed, caches warm; no lingering errors.
- Any manual failover reversed in a planned, gradual way.

---

## Prevention

- Synthetic checks per critical dependency, from your side.
- Subscribe to provider status feeds and account health events (webhooks into the incident channel).
- Contracts/SLAs with vendors; escalation contacts.
- Regional failover game days (topic 88).

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: The trigger was the provider's outage, which we can't control. Our root cause is how much it hurt us: e.g. all checkout depended on one payment provider with no fallback and no timeout, and our status page was hosted on the same CDN.

2. **Q: How did you detect it? Why not earlier?**
   A: Our SLO alert fired before the provider's status page changed. Earlier detection would need per-dependency synthetic checks.

3. **Q: What did you check first?**
   A: Our error breakdown by dependency, to separate "us" from "them" within minutes.

4. **Q: How did you stop impact quickly?**
   A: Switched card payments to the secondary provider via flag and disabled non-essential features that depended on the failed one.

5. **Q: Permanent fix?**
   A: Automatic provider failover via circuit breaker, a status page hosted independently, and a dependency inventory with fallbacks.

6. **Q: How did you confirm?**
   A: Checkout success back in SLO on the secondary provider; after recovery, a planned switch back.

7. **Q: What if the failover itself failed?**
   A: Then degrade further (queue orders for later processing, show clear messages) and communicate honestly. And in the postmortem: an untested failover is not a failover — schedule regular tests.

8. **Q: Should every company be multi-region or multi-cloud?**
   A: No. It's expensive and complex, and badly done it adds failure modes. Start with good multi-AZ design, static stability, graceful degradation and clear communication; add multi-region for the parts where the business impact justifies it.

9. **Q: What would you do differently?**
   A: Map hidden dependencies, especially incident tooling, and test operating while the main provider is impaired.

10. **Q: What if the cloud control plane was down but your instances kept running?**
    A: Don't make changes that need the control plane (scaling in, deploys, replacing instances) — you might not get capacity back. Keep running on the data plane, and wait for control plane recovery before resuming changes.
