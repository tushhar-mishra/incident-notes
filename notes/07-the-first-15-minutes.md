# 07 · The First 15 Minutes

> **Level:** ★★ intermediate
>
> **You will learn:** a calm, repeatable **script** for the first 15 minutes after a page: acknowledge, **scope the impact**, ask **"what changed?"**, pick the **fastest safe mitigation** from the **toolbox** (rollback, feature flag, failover, traffic shift, scale, shed load, restart), **one change at a time**, how to decide **rollback vs fix forward**, and how to **keep evidence** before destructive steps.
>
> **You need:** topics 01–06.
>
> **Part 1 · Incident basics** · Previous: [06 · Incident communication](06-incident-communication.md) · Next: [08 · Debugging under pressure](08-debugging-under-pressure.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Most outages are caused by a **change**. Most are fixed fastest by **undoing that change**. So the first 15 minutes are about **impact**, **what changed**, and the **fastest safe mitigation**, not about deep root cause.

```
0–2 min    ACK + open channel. "I'm on it."
2–5 min    IMPACT: what do users see? how many? since when? getting worse?
5–8 min    WHAT CHANGED? deploys, config, flags, infra (Terraform/Ansible), traffic, dependencies, certs, time
8–12 min   MITIGATE: fastest safe action (usually ROLL BACK the suspect change)
12–15 min  VERIFY with the user-facing metric. Communicate. Declare/upgrade if needed.
```

**Picture it.** You come home and the kitchen floor is wet. You don't start by studying the plumbing. You **turn off the water** (mitigate), **then** find the leak.

**The one thing to know.** **If something changed just before the problem started, undo it first, even if you're not sure it's the cause.** A rollback is usually fast, safe and reversible. Debugging under pressure is slow.

**Try it now:** write your team's top five "big red buttons": the exact command or click for rollback, flag off, failover, traffic shift and scale-up.

**You've got it if you can:** walk through the 15-minute script for "5xx errors jumped from 0.1% to 8% at 14:03".

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

A **triage script**: a short, fixed sequence of questions and actions for the start of any incident. Using the same script every time means you don't have to think about **how** to start, so your brain is free to think about **what** is wrong.

---

## Why it exists

Under stress, people tend to:
- Jump to the first theory and stick to it (**tunnel vision**).
- Start reading code or logs deeply while users are still down.
- Make several changes at once, then not know which one helped.
- Forget to tell anyone.

A script prevents all four.

---

## Analogy

**First aid: "DR ABC".** First-aiders learn a fixed order: Danger, Response, Airway, Breathing, Circulation. They don't diagnose the disease. They keep the person alive until the hospital takes over. The first 15 minutes are your DR ABC.

---

## How it works

### Step 1: Acknowledge (0–2 min)

- Ack the page, so it doesn't escalate.
- Open (or join) the incident channel. Post: "Ack'd, looking. Impact unknown yet."

### Step 2: Impact (2–5 min)

Look at the **user-facing** dashboards first (golden signals, topic 08):

| Question | Where to look |
|---|---|
| What are users seeing? Errors? Slowness? Wrong data? | SLO dashboard, load balancer 5xx, synthetic checks |
| How many / which ones? | Error rate %, by region / endpoint / customer |
| Since when, exactly? | The graph: find the **first minute** it went bad |
| Getting worse, stable, or recovering? | Trend over last 15 min |

Now set the **severity** (topic 02).

### Step 3: What changed? (5–8 min)

The exact start time from step 2 is your key. Look for **anything** that changed just before it:

| Type of change | Where to check |
|---|---|
| **Code deploy** | CI/CD history (Jenkins, GitHub Actions, Argo CD), `kubectl rollout history` |
| **Config / feature flag** | Flag tool audit log, ConfigMap changes, config repo commits |
| **Infrastructure** | Terraform apply logs, Ansible runs, cloud console audit (CloudTrail) |
| **Platform** | Kubernetes upgrade, node pool change, OS auto-updates (topic 25) |
| **Traffic** | Request rate graph: spike? new client? bot? marketing email sent? |
| **Dependency** | Status pages of cloud provider, payment provider, CDN, DNS provider |
| **Time-based** | Certificate expiry (topic 27), cron jobs, end of month, leap second, daylight saving |
| **Data** | A big batch job, a migration, a customer importing 10 million rows |

```bash
# Kubernetes: what rolled out recently?
kubectl get deploy -A --sort-by=.metadata.creationTimestamp | tail
kubectl rollout history deploy/checkout -n shop
kubectl get events -A --sort-by=.lastTimestamp | tail -30

# Linux host: what changed?
sudo journalctl --since "30 min ago" -p warning
rpm -qa --last | head        # RHEL: recently installed packages
grep " install \| upgrade " /var/log/dpkg.log | tail   # Debian/Ubuntu

# Git: what merged recently?
git log --since="3 hours ago" --oneline --merges
```

> **3 AM tip:** if you find **no** change, widen the window (a change can take hours to show up, e.g. a memory leak or a cache expiring) and look at **traffic** and **dependencies**.

### Step 4: Mitigate (8–12 min)

**The mitigation toolbox**, fastest and safest first:

| # | Action | Use when | Example |
|---|---|---|---|
| 1 | **Roll back** the change | A deploy/config/flag change lines up with the start | `kubectl rollout undo deploy/checkout`; revert in Argo CD; re-run previous pipeline |
| 2 | **Feature flag off / kill switch** | A new feature is misbehaving | Turn off `new_pricing_engine` |
| 3 | **Shift traffic** away from the bad part | One region / AZ / cell / node pool is bad | DNS weights, load balancer, `kubectl cordon` |
| 4 | **Fail over** | A primary component died (DB, region) | Promote replica; regional failover |
| 5 | **Scale up / out** | Real capacity shortage | `kubectl scale`, bigger instance. ⚠️ Won't help if the bottleneck is downstream |
| 6 | **Shed load / rate limit** | Overload, retry storm, abusive client | Block a client at the WAF; lower rate limits; turn off non-critical features |
| 7 | **Restart** | Stuck process, memory leak | `kubectl rollout restart` — capture evidence first |
| 8 | **Fix forward** | Rollback is impossible or riskier | Hotfix through the pipeline, with a second reviewer |

**Rules:**
- **One change at a time.** Announce it: "Rolling back checkout to v2.13 now." Wait and watch before the next one.
- **Time-box.** "If the error rate hasn't dropped 5 minutes after the rollback, we fail over."
- **Prefer reversible actions.** Rollback can be rolled forward again. Deleting data cannot be undone.

### Rollback vs fix forward

| Roll back when… | Fix forward when… |
|---|---|
| The change is recent and rollback is tested | A database migration already ran and old code can't read the new schema |
| You're not sure what's wrong | The fix is a one-line, obvious, reviewed change and rollback is slower |
| It's the middle of the night | The bad version fixed a security hole that rollback would reopen |

The default is **roll back**. Fix forward needs a reason.

### Keep evidence before destructive steps

Restarting a pod or a server **destroys** the evidence (memory state, open connections, local logs). If it costs only a minute, grab it first:

```bash
kubectl logs deploy/checkout -n shop --all-containers --since=30m > /tmp/inc-logs.txt
kubectl describe pod -n shop -l app=checkout > /tmp/inc-describe.txt
kubectl get events -n shop --sort-by=.lastTimestamp > /tmp/inc-events.txt
# for a JVM: jcmd <pid> Thread.print / GC.heap_dump (if safe and fast)
```

For **security** incidents, don't restart or wipe at all until the security team says so.

### Step 5: Verify and communicate (12–15 min)

- Did the **user-facing** metric recover? (Not "the pod is Running"; the **error rate**.)
- Post an update (topic 06) with the next update time.
- If not recovered: back to step 3 with the next hypothesis, or the next tool in the box.

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Root-causing first** | 45 minutes reading code while a rollback was available | Script: mitigation step at minute 8 |
| **Tunnel vision** | "It must be the database" for an hour | IC asks "what would make us wrong?"; list 2–3 hypotheses |
| **Shotgun changes** | Restart + scale + rollback at once; nobody knows what worked | One change at a time, announced |
| **Rollback doesn't exist or is broken** | "We can't roll back because…" | Practise rollback; keep N previous versions deployable |
| **Declaring victory early** | Pods are Running, users still failing | Verify with the SLI, not with component health |
| **"Nothing changed"** | No obvious deploy | Widen the window; check flags, infra, traffic, deps, certs, time |

---

## Where you'll use it

- Every scenario in Parts 2 and 4 starts with this script.
- **Kubernetes rollback:** [K8s 25 · Deployments](../../CKA-Certification-Course-2025/notes/25-deployments.md) (`rollout undo`, history).
- **Deployment strategies** that make step 4 easy: [SD 34](../../System-Design/notes/34-deployment-strategies.md) (canary, blue/green, flags).
- **Terraform:** "what changed?" includes applies: [Terraform 66](../../Terraform/notes/66-production-scenarios.md).

---

## Quick recap

- 0–2 ack · 2–5 impact · 5–8 what changed · 8–12 mitigate · 12–15 verify + communicate.
- Most incidents follow a change; undo it first.
- Toolbox: rollback, flag off, shift traffic, fail over, scale, shed load, restart, fix forward (last).
- One change at a time, announced, time-boxed.
- Grab cheap evidence before restarting. Verify with the user-facing metric.

### Interview questions

1. **Q: You're paged: 5xx errors went from 0.1% to 8% at 14:03. Walk me through your first 15 minutes.**
   A: Ack and open the incident channel. Check the SLO dashboard: 8% errors, all regions, started 14:03, stable. Declare SEV2 (or SEV1 if it's checkout). Then "what changed?": deploy history, flag audit log, Terraform/infra changes, traffic and dependency status around 14:00. If a deploy went out at 14:01, I roll it back immediately, announce it, and watch the error rate. If it recovers, post an update and keep monitoring; root cause comes after. If not, I move to the next hypothesis (dependency, traffic) and the next tool (fail over, shed load).

2. **Q: When would you fix forward instead of rolling back?**
   A: Only when rollback is impossible or more dangerous: for example a schema migration that old code can't handle, or a rollback that would reopen a security hole. Otherwise rollback is the default because it's fast, tested and reversible.

3. **Q: Nothing was deployed. Where do you look?**
   A: Other kinds of change: config and feature flags, infrastructure applies, platform upgrades and OS auto-updates, certificate expiry, cron jobs, traffic changes (spikes, new clients, bots), and dependency outages. I also widen the time window, because slow problems like leaks or cache expiry show up hours after their cause.

4. **Q: Why "one change at a time"?**
   A: So we know what worked. If we roll back, scale up and restart together and the problem goes away, we don't know the cause, and if it gets worse, we don't know which change hurt. It also prevents two people from fighting each other's changes.
