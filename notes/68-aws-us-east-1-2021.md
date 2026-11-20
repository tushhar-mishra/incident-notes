# 68 · AWS us-east-1, December 2021: Clients That Didn't Back Off

> **Level:** ★★★ advanced
>
> **Pattern:** An **automated scaling activity** triggers unexpected behaviour in many **internal clients** → a **connection surge** overwhelms the devices between AWS's internal network and its main network → **retries without proper backoff** keep it congested → **monitoring blind**, deployment tools impaired → hours of degraded control planes. **Families:** automation · retry storm (topic 36) · cascading failure (topic 38) · monitoring sharing fate (topic 44).
>
> **You will learn:** how AWS separates an **internal network** (monitoring, internal DNS, authorization, parts of the EC2 control plane) from the main AWS network, how a scaling event set off a **connection storm** between them, why a **latent issue** stopped clients from backing off, why diagnosis was slow (**real-time monitoring unavailable**), which services were and weren't affected (running instances fine; APIs, launches, Connect, STS impaired), and the fixes.
>
> **You need:** Part 1, topics 36, 38, 39, 44.
>
> **Part 3 · Real outages · Capacity and overload** · Previous: [67 · Slack 2021](67-slack-2021.md) · Next: [69 · Canva 2024](69-canva-2024.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** AWS runs an **internal network** for its own foundational services (monitoring, internal DNS, authorization, parts of the EC2 control plane), connected to the main AWS network through networking devices. At 7:30 AM PST on 7 December 2021, an **automated activity to scale** a service in the main network triggered **unexpected behaviour from a large number of clients** in the internal network: a **surge of connection activity** overwhelmed the connecting devices. Latency and errors caused **more retries**, and a **latent issue** stopped clients from backing off. Congestion persisted for hours.

```
7:30 AM automated scaling → many internal clients open connections at once
   → devices between internal and main network overwhelmed → latency + errors
   → clients retry; latent bug: they don't back off → congestion persists (metastable, topic 36)
   → real-time monitoring impaired → operators work from logs
   → EC2 APIs/launches, Route 53 APIs, STS, Console, API Gateway, Connect, EventBridge, Fargate impaired
   → running EC2 instances and most customer workloads NOT directly affected
9:28 AM internal DNS moved off congested paths · 2:22 PM network devices fully recovered
```

**Picture it.** A single bridge connects a company's back office to its shop floor. One morning, a routine change makes hundreds of back-office staff all phone the shop floor at once. The bridge's switchboard jams. Each caller keeps redialling immediately. The security cameras that would show what's happening run over the same switchboard.

**The one thing to know.** **Backoff is a safety feature, not an optimisation.** If clients don't back off, a brief overload becomes a long one — and you'll often be blind while it happens.

**You've got it if you can:** explain why the problem persisted after the trigger, and why diagnosis was slow.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 7 December 2021, an automated activity to scale capacity of an AWS service in the main network triggered unexpected behaviour from a large number of clients inside AWS's internal network, causing a surge of connection activity that overwhelmed the networking devices between the two networks. Retries and a latent issue that prevented clients from backing off sustained the congestion, impairing monitoring and many AWS control-plane APIs in us-east-1 for several hours. Running EC2 instances were unaffected; network devices fully recovered at 2:22 PM PST, with some services recovering later.

## Impact

| Item | Detail (from AWS's summary, PST) |
|---|---|
| Not affected | Running EC2 instances; Lambda APIs and invocations "operated normally"; direct S3/DynamoDB access (VPC endpoint access was impaired) |
| EC2 | API errors/latency from 7:33 AM; new instance launches recovered 2:40 PM (affecting RDS, EMR, WorkSpaces resource creation) |
| Route 53 | API changes impaired 7:30 AM–2:30 PM (existing DNS answers unaffected) |
| Console / STS | Console login failures until 2:22 PM; STS fully recovered 4:28 PM |
| API Gateway / Connect | API Gateway largely recovered 4:37 PM; Connect normal 4:41 PM |
| EventBridge | Delivery disabled 12:35–2:35 PM to reduce load; latency until 6:40 PM |
| Fargate / ECS / EKS | API errors; Fargate normal 5:00 PM (some capacity errors for hours) |
| CloudWatch | Monitoring delays; a small amount of data not captured |
| Support / status | Support case creation impacted 7:33 AM–2:25 PM; Service Health Dashboard failover impaired; updates via a global banner that some customers found hard to see |
| Customers / cost | **Not publicly disclosed** |

## Timeline (PST, 7 December 2021)

| Time | Event |
|---|---|
| 7:30 AM | Automated scaling activity → connection surge → congestion |
| 7:33 AM | EC2 API errors and latency begin; support case creation impacted |
| 8:22 AM | AWS successfully updating the Service Health Dashboard |
| 9:28 AM | Internal DNS traffic moved away from congested paths; DNS resolution errors fully recovered (congestion not yet resolved) |
| 12:35 PM | EventBridge event delivery disabled to reduce load |
| 1:15 PM | EC2 API error rates improving (except launches) |
| 1:34 PM | Congestion significantly improved |
| 2:22 PM | All network devices fully recovered |
| 2:35 PM | EventBridge delivery re-enabled |
| 2:40 PM | EC2 instance launches recovered |
| 4:28–5:00 PM | STS, API Gateway, Connect, Fargate recover |
| 6:40 PM | EventBridge latency ends |

## How it happened

```
 ┌────────── internal network ──────────┐        ┌───────── main AWS network ─────────┐
 │ monitoring, internal DNS, authz,      │        │ customer-facing services            │
 │ parts of EC2 control plane, clients   │◄──────►│ (one service scales automatically) │
 └───────────────────────────────────────┘ devices└─────────────────────────────────────┘
                                          (routing + NAT)
 7:30 AM: scaling activity → unexpected behaviour from MANY internal clients → connection surge
 devices overwhelmed → latency/errors → clients retry → more connections
 "a latent issue prevented these clients from adequately backing off" → persistent congestion
```

AWS noted the client code path had been in production for many years; the scaling activity triggered a "previously unobserved behavior". The specific service and client behaviour are **not publicly disclosed**.

## Why it spread

- **Shared chokepoint:** many internal services communicate across the same devices.
- **No effective backoff** → metastable congestion (topic 36).
- **Blind operators:** congestion "immediately impacted the availability of real-time monitoring data".
- **Impaired tools:** internal deployment systems were also affected, slowing remediation; operators were deliberately cautious to avoid harming workloads that were still working.
- **Dependent services:** API Gateway servers needed recycling, blocked until EC2 APIs recovered; Connect depended on API Gateway.

## How they found it

Operators relied on **logs** because real-time monitoring was unavailable. They first identified elevated **internal DNS errors** and moved DNS traffic away (9:28 AM) — this fixed DNS errors but not the congestion. They then worked with reduced visibility to identify top traffic sources.

## What they did to fix it

1. Moved internal DNS traffic away from congested paths.
2. Identified top traffic sources and isolated them to dedicated network devices; disabled some heavy network traffic services.
3. Brought additional networking capacity online.
4. Disabled EventBridge event delivery temporarily to reduce load.
5. Made changes cautiously to protect working workloads.

## Permanent fix (as listed by AWS)

| Change | Pattern it addresses |
|---|---|
| **Disabled the scaling activities** that triggered the event until all remediations are deployed | Trigger |
| Fix for the **latent backoff issue** (expected within two weeks of the report) | Retry storm |
| Additional network configuration protecting potentially impacted networking devices | Chokepoint protection |
| API Gateway: keep servers healthy when the internal network is unavailable; better recycling | Dependency resilience |
| New Service Health Dashboard version (early 2022) to make impact easier to understand | Communication |
| New support system architecture running across multiple AWS regions | Support availability |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Many clients behind one NAT gateway / proxy / service mesh egress | Capacity and connection limits on chokepoints; alerts on connection rate |
| Client libraries with retries but broken or missing backoff | Standard library with exponential backoff + jitter + budget; chaos tests (topic 36) |
| Autoscaling events that make many clients reconnect at once | Jittered reconnects; connection pooling; slow-start |
| Monitoring traffic sharing the congested path | Separate monitoring paths; prioritised telemetry (topic 44) |
| Deployment tools on the same network as production | Independent deploy/break-glass paths |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A latent issue that prevented internal clients from backing off, so a connection surge triggered by an automated scaling activity became persistent congestion on the devices linking the internal and main networks.

2. **Q: How was it detected, and why was diagnosis slow?**
   A: Impact was immediate; diagnosis was slow because real-time monitoring ran over the congested network, so operators worked from logs.

3. **Q: What was checked first?**
   A: Internal DNS errors (visible in logs); fixing DNS helped DNS but showed the congestion was the bigger problem.

4. **Q: How did they stop the impact?**
   A: Moved and isolated traffic, disabled heavy sources, added network capacity — cautiously, to avoid disrupting working workloads.

5. **Q: What changed permanently?**
   A: Triggering scaling disabled, backoff bug fix, device protections, API Gateway resilience, a new status dashboard and multi-region support system.

6. **Q: How would you confirm a backoff fix?**
   A: Fault-inject latency/errors on the chokepoint and verify client retry rates fall rather than rise.

7. **Q: What if isolating traffic sources hadn't helped?**
   A: Shed load more aggressively (disable more non-critical internal services), or add capacity faster; recovery only begins when load drops below capacity.

8. **Q: How would you prevent this class of incident?**
   A: Backoff + jitter everywhere, connection limits at chokepoints, monitoring on an independent path, and testing large automated scaling events against client behaviour.

9. **Q: Why were running customer workloads mostly fine?**
   A: The data plane (running instances) didn't depend on the congested path; the control plane (APIs, launches) did. That's the static-stability idea from topic 39.

10. **Q: What should a customer have done differently?**
    A: Avoid needing control-plane actions (launching instances, changing DNS) to survive an incident: pre-provision capacity, use static configuration, and design multi-AZ/multi-region for critical paths.

## Source

- AWS, "Summary of the AWS Service Event in the Northern Virginia (US-EAST-1) Region" (December 2021): <https://aws.amazon.com/message/12721/>
