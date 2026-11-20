# 62 · AWS us-east-1, October 2025: The Empty DNS Record

> **Level:** ★★★ advanced
>
> **Pattern:** A **race condition** in **DNS automation** → DynamoDB's regional endpoint gets an **empty DNS record** → everything that needs DynamoDB fails → recovery triggers **congestive collapse** in EC2's internal systems → **NLB health checks** remove healthy capacity. **Families:** automation gone wrong · DNS · cascading failure and metastable recovery (topics 36, 38) · dependency (topic 39).
>
> **You will learn:** how DynamoDB's DNS **Planner** and three independent DNS **Enactors** worked, the exact race that let a **delayed Enactor apply an old plan** and a **cleanup job delete it** — removing all IPs for `dynamodb.us-east-1.amazonaws.com`; why EC2 launches kept failing **hours after** DynamoDB recovered (DWFM leases, congestive collapse); why NLB health checks made things worse; and AWS's fixes (automation disabled worldwide, velocity controls, recovery test suites).
>
> **You need:** Part 1, topics 26, 29, 36, 38, 39.
>
> **Part 3 · Real outages · DNS and network** · Previous: [61 · Cloudflare 1.1.1.1 2025](61-cloudflare-1111-2025.md) · Next: [63 · Datadog 2023](63-datadog-2023.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** DynamoDB's DNS records are managed by automation: a **Planner** creates DNS plans; three **Enactors** (in different AZs) apply them. One Enactor was **slow**. Meanwhile, another Enactor applied **newer** plans and then ran **cleanup** of old plans. The slow Enactor then applied its **old** plan over the newer one; cleanup deleted that old plan as obsolete — and with it **all IP addresses** for the regional DynamoDB endpoint. The record was **empty**, and the system was stuck in a state its own automation couldn't repair.

```
Planner ──► plan v100, v101, v102 ...
Enactor B (fast): applies v110 → cleanup deletes plans much older than v110
Enactor A (slow, stuck retrying): finally applies v100 OVER v110   (its "newer than current?" check was stale)
Cleanup: "v100 is ancient → delete" → regional record now has NO IPs → empty DNS answer
11:48 PM PDT Oct 19: DNS failures for dynamodb.us-east-1 → DynamoDB unreachable
 2:25 AM: DNS restored (manual) → 2:40 AM DynamoDB recovered
 then: EC2 DWFM congestive collapse → launches fail until ~10:37 AM, network config backlog
       NLB health checks flap → capacity removed → connection errors 5:30 AM–2:09 PM
 2:20 PM PDT Oct 20: event ends
```

**Picture it.** Three workers update the shop's sign with today's prices. One worker is stuck in traffic with yesterday's sign. Another puts up today's sign and throws away old signs. The late worker arrives and hangs yesterday's sign over today's. The cleaner then throws away "yesterday's sign" — the one now on the wall. The wall is blank, and customers think the shop is gone.

**The one thing to know.** **Automation needs the same safety as humans:** check freshness **at the moment of applying**, never leave a critical record **empty**, and have a **tested manual recovery** for when automation gets itself stuck.

**You've got it if you can:** explain the race step by step and why EC2 was impacted for hours after DynamoDB recovered.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 19–20 October 2025, a latent race condition in DynamoDB's automated DNS management left the regional endpoint `dynamodb.us-east-1.amazonaws.com` with an empty DNS record, causing DynamoDB API failures in us-east-1 from 11:48 PM to 2:40 AM PDT. Services depending on DynamoDB failed, and EC2's instance-management system then entered congestive collapse while recovering, delaying new instance launches; Network Load Balancer health checks, confused by delayed network configuration, removed healthy capacity. AWS declared the event over at 2:20 PM PDT on 20 October.

## Impact

| Item | Detail (from AWS's summary) |
|---|---|
| DynamoDB | API errors 11:48 PM (Oct 19) – 2:40 AM (Oct 20) PDT; global tables replicas in other regions available but replication with us-east-1 lagged |
| EC2 | New launch failures/"insufficient capacity" and connectivity issues for new instances until 1:50 PM; running instances unaffected |
| NLB | Increased connection errors 5:30 AM – 2:09 PM |
| Others (examples) | Lambda (to 2:15 PM), ECS/EKS/Fargate (to 2:20 PM), Amazon Connect (to 1:20 PM), STS (to 9:59 AM), IAM console sign-in (to 1:25 AM), Redshift (clusters fully restored 4:05 AM Oct 21), AWS Support case handling (to 2:40 AM), and more |
| Overall window | 11:48 PM Oct 19 – 2:20 PM Oct 20 PDT (~14.5 hours) |
| Number of customers / cost | **Not publicly disclosed** |

## Timeline (PDT)

| Time | Event |
|---|---|
| Oct 19, 11:48 PM | Race leaves DynamoDB regional endpoint DNS record empty; systems connecting to DynamoDB get DNS failures. EC2 DWFM state checks (which depend on DynamoDB) start failing |
| Oct 20, 12:38 AM | Engineers identify DynamoDB's DNS state as the source |
| 1:15 AM | Temporary mitigations let some internal services reach DynamoDB; key internal tooling repaired |
| 2:25 AM | All DNS information restored (manual operator intervention) |
| 2:25–2:40 AM | Customers recover as cached DNS records expire; DynamoDB recovery complete at 2:40 AM |
| 2:25 AM → | DWFM tries to re-establish leases with all droplets (servers); falls into **congestive collapse** |
| 4:14 AM | Engineers throttle incoming work and selectively restart DWFM hosts |
| 5:28 AM | DWFM has leases with all droplets; launches start succeeding (with throttling); Network Manager processes a backlog of network configuration |
| 5:30 AM | NLB connection errors begin (health checks fail for instances whose network state hasn't propagated) |
| 6:52 AM | NLB monitoring detects the health-check issue |
| 9:36 AM | Automatic NLB health-check failover disabled → healthy capacity returns |
| 10:36–10:37 AM | Network propagation normal; launches succeed |
| 11:23 AM | EC2 throttles start relaxing |
| 1:50 PM | EC2 fully recovered |
| 2:09 PM | NLB automatic DNS health-check failover re-enabled |
| 2:20 PM | Event ends (ECS/EKS/Fargate recovered) |

## How it happened

```
                    ┌───────────────┐   plans (newer and newer)
                    │  DNS Planner  │──────────────────────────────┐
                    └───────────────┘                               ▼
      ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
      │ DNS Enactor (AZ1)│  │ DNS Enactor (AZ2)│  │ DNS Enactor (AZ3)│   (independent, for resilience)
      └────────┬─────────┘  └────────┬─────────┘  └──────────────────┘
   delayed, retrying      fast: applies newest plan,
   with an OLD plan       then cleanup deletes plans many generations old
               │                     │
               └──► applies OLD plan over the NEWER one (its "is my plan newer?" check was done earlier → stale)
                                     │
                        cleanup deletes that OLD plan → regional endpoint loses ALL IP addresses
                        → system left inconsistent → no Enactor can apply further plans
                        → needs manual operator intervention
```

### Then: EC2 and NLB (the long tail)

- **DWFM** (DropletWorkflow Manager) keeps a **lease** on each physical server ("droplet"); its health checks depend on DynamoDB. While DynamoDB was down, leases timed out. After recovery, DWFM had to re-establish leases on a huge number of droplets at once and entered **congestive collapse** — it couldn't make progress — and AWS says there was no established operational recovery procedure for this. Throttling plus restarting DWFM hosts cleared the queues.
- **Network Manager** then had a backlog of network configuration to propagate; new instances launched without connectivity until it caught up.
- **NLB** health checks brought new instances into service before their network state had propagated, so checks failed, then passed, then failed — removing and re-adding capacity, overloading the health-check subsystem and triggering **automatic AZ DNS failover** that took capacity out of service.

## Why it spread

- **DynamoDB is a dependency of many AWS services** (including EC2's own control-plane components, IAM/STS paths, Lambda, Connect) in us-east-1.
- **DNS caching** delayed recovery until caches expired (2:25–2:40 AM).
- **Metastable recovery** (topic 36): the mass re-lease work overloaded DWFM.
- **Automation amplifying a symptom:** NLB health-check failover removed healthy capacity.

## How they found it

AWS says engineering teams for impacted services were **immediately engaged**; by 12:38 AM they had identified DynamoDB's DNS state as the source. NLB monitoring detected its issue at 6:52 AM. The exact first alarm for the DNS fault is **not publicly disclosed**.

## What they did to fix it

1. Temporary mitigations so internal services and tooling could reach DynamoDB (1:15 AM).
2. **Manually repaired** the DNS state (DNS restored 2:25 AM).
3. For EC2: **throttled incoming work**, **selectively restarted DWFM hosts**, then let Network Manager drain its backlog; relaxed throttles gradually.
4. For NLB: **disabled automatic health-check failover** (9:36 AM), returning healthy capacity; re-enabled after EC2 recovered.
5. For Lambda: throttled event source mappings and async invocations to prioritise synchronous calls.

## Permanent fix (as listed by AWS)

| Change | Pattern it addresses |
|---|---|
| **DynamoDB DNS Planner and Enactor automation disabled worldwide** until the race is fixed and protections against applying incorrect DNS plans are added | Automation acting on stale data |
| NLB **velocity control** limiting how much capacity a single NLB can remove during health-check-driven AZ failover | Automation amplifying failures |
| EC2: an **additional test suite** exercising the DWFM recovery workflow at scale | Untested recovery paths |
| EC2: better throttling that **rate-limits incoming work based on queue size** | Congestive collapse |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| external-dns / Terraform / scripts managing DNS records concurrently | One writer per record (locking), compare-and-swap on versions, refuse to publish an **empty** record |
| Cleanup jobs deleting "old" objects (images, snapshots, plans, ConfigMaps) | Never delete what's currently in use; check references at deletion time |
| Kubernetes controllers recovering after an outage (relist storms, mass reconciles) | Rate-limited work queues, backoff, priority (API Priority and Fairness) (topic 22) |
| LB/health-check automation that can remove all capacity | Fail-open thresholds, max % removable (topic 29) |
| Everything in one region (and all of AWS's own region-scoped services) | Know your us-east-1 dependencies; static stability; multi-region for critical paths (topic 39) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A latent race condition in DNS automation: a delayed Enactor's freshness check was stale when it applied an old plan, and a cleanup process then deleted that plan, leaving the regional record empty and the system unable to self-correct.

2. **Q: How was it detected, and why didn't automation fix it?**
   A: Teams engaged immediately and found the DNS state in under an hour. The inconsistent state blocked all Enactors from applying further plans, so manual intervention was required.

3. **Q: What would you check first as a customer?**
   A: `dig dynamodb.us-east-1.amazonaws.com` — an empty/failed answer immediately explains the errors and points to AWS, not your code.

4. **Q: How did AWS stop the impact?**
   A: Manual DNS repair, then throttling and host restarts for EC2's DWFM, then disabling NLB health-check failover.

5. **Q: Why did EC2 problems last until the afternoon?**
   A: Recovery itself overloaded DWFM (congestive collapse), then a network configuration backlog delayed connectivity for new instances, and NLB health checks reacted badly to that delay.

6. **Q: What changed permanently?**
   A: The DNS automation disabled worldwide pending fixes, NLB velocity controls, a DWFM recovery test suite, and queue-aware throttling.

7. **Q: What if manual DNS repair had failed?**
   A: Clients could only wait (or use other regions); customers with multi-region designs (e.g. global tables) could fail over. That's the case for knowing your regional dependencies.

8. **Q: How would you prevent this class of incident?**
   A: Freshness checks at apply time, never publishing empty records, safe cleanup, and practised recovery from "automation stuck" states.

9. **Q: What would you do differently as an AWS customer?**
   A: Avoid hard dependencies on a single region for critical paths, design statically stable systems that don't need to launch instances during an outage, and cache configuration/credentials.

10. **Q: What does this teach about recovery at scale?**
    A: Recovery is a load event. Systems need throttled, prioritised, tested recovery paths — otherwise fixing the first problem triggers the next.

## Source

- AWS, "Summary of the Amazon DynamoDB Service Disruption in the Northern Virginia (US-EAST-1) Region": <https://aws.amazon.com/message/101925/>
