# 91 · "Tell Me About a Production Incident You Handled"

> **Level:** ★★ intermediate
>
> **You will learn:** how to answer the most common senior DevOps/SRE behavioural question with the **STAR** method (**S**ituation, **T**ask, **A**ction, **R**esult), what interviewers are really scoring, a **reusable template**, three **senior-level example answers** (a deploy-caused outage, a capacity/cascade incident, a security incident), how to handle **follow-up drilling**, the **traps** that make good engineers sound junior, and how to build your story bank — honestly.
>
> **You need:** Parts 1–4.
>
> **Part 5 · Interview readiness** · Previous: [90 · Capstone game day](90-drill-game-day-capstone.md) · Next: [92 · Interview question bank](92-interview-question-bank.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Interviewers aren't checking whether you've seen a dramatic outage. They're checking **how you think and behave** under pressure: do you protect users first, coordinate, communicate, find real root causes, and make the system better afterwards?

```
S  Situation  (15%)  context: system, scale, when, what users saw — numbers
T  Task       (10%)  YOUR role and responsibility (on-call? IC? owner?)
A  Action     (50%)  what YOU did, in order: triage → mitigate → communicate → root cause
R  Result     (25%)  impact numbers, time to mitigate, what changed permanently, what you learned
```

**The one thing to know.** Spend **half your time on Action**, say **"I"** for what you did, show **mitigation before root cause**, and end with **systemic, measurable** improvements.

**Try it now:** write one incident from your experience (or a lab drill) in the template below, then say it aloud in under three minutes.

**You've got it if you can:** tell a 2–3 minute STAR story and survive five follow-up "why" questions.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

**STAR** structures a story so the listener can follow it and score it:

| Part | Questions it answers | Senior signals |
|---|---|---|
| **Situation** | What system? What scale? What broke, from the user's view? | Quantified impact (%, users, revenue path), SLO language |
| **Task** | What was your role? | Ownership: on-call, IC, service owner |
| **Action** | What did you do, step by step, and why? | Mitigate-first, hypotheses, coordination, comms, one change at a time |
| **Result** | How did it end? What changed afterwards? | MTTM/MTTR numbers, postmortem actions that **prevent a class** of failure, follow-through |

---

## Why it exists

Behavioural questions predict behaviour: how you acted in the past is the best guide to how you'll act on their pager. For senior roles they want evidence of **judgement**, **leadership without authority**, and **systems thinking**.

What interviewers score (typical rubric):
1. **Impact awareness:** did you understand and quantify user/business impact?
2. **Prioritisation:** mitigation before root cause?
3. **Technical depth:** sound debugging, real root cause (not just the trigger)?
4. **Collaboration and communication:** roles, updates, stakeholders?
5. **Learning:** blameless, systemic fixes, follow-through, measurable improvement?
6. **Ownership and honesty:** clear about what you did vs the team; admits mistakes.

---

## Analogy

A pilot debriefing after an emergency landing: what happened, what they were responsible for, each decision and why, and what the airline changed afterwards. Calm, specific, no blame, no drama.

---

## How it works

### Template (fill this in for 3–5 incidents)

```
SITUATION: At <company/team>, I was on call for <service> (<scale: req/s, users, revenue role>).
           At <time>, <symptom users saw> — <metric: e.g. checkout errors 25%, p99 4 s> — starting <when>.
TASK:      As <on-call / IC / owner>, I was responsible for <restoring service / coordinating / fixing>.
ACTION:    1. Triage: confirmed impact via <SLI/dashboard>; declared <SEV>; opened channel; assigned <roles>.
           2. What changed: found <deploy/config/traffic/dependency> at <time>.
           3. Mitigation: <rollback/failover/shed/flag> because <reason: fast, reversible>; result <metric>.
           4. Communication: <updates every N min, status page, stakeholders>.
           5. Root cause: <method: logs/traces/EXPLAIN/...> → <root cause> + <contributing factors>.
RESULT:    Mitigated in <N> min, resolved in <M>; impact <numbers>.
           Postmortem: <2–3 systemic actions> → <measurable outcome: e.g. no repeat in 12 months,
           change failure rate from X to Y, MTTD from 20 to 3 min>. I learned <one honest lesson>.
```

### Example 1 — Deploy-caused outage (classic, strong)

> **S:** I was on the payments platform team at an e-commerce company handling about 2,000 checkout requests per second at peak. One Friday at 18:05, our burn-rate alert fired: checkout errors had jumped from 0.1% to about 25%, meaning roughly one in four customers couldn't pay.
>
> **T:** I was the primary on-call, so I owned the response until an incident commander joined.
>
> **A:** First I confirmed impact on the SLO dashboard and in load balancer logs — it was all regions and growing — so I declared a SEV1, opened the incident channel and paged our IC rota. While waiting I checked what had changed: payments-svc v2.14 had gone to 100% at 17:58, and our payment provider's status page showed "degraded" from 18:00, so I had two candidates. Because rollback was fast and reversible, I rolled back v2.14 immediately and announced it. Errors dropped to about 6% within eight minutes, which told me the deploy was most of the problem but not all. The IC took over coordination and comms — status page every 15 minutes — and I focused on the remaining errors. Traces showed calls to the provider timing out, and v2.14 had added three retries without backoff, so even after rollback, queued retries were still hitting a degraded provider. We enabled our secondary provider for 50% of traffic via a feature flag, which brought errors under 0.5% by 18:50.
>
> **R:** Customer impact was 52 minutes, with about 45 minutes at significant error rates. In the blameless postmortem we found three contributing factors: retries without backoff, a 100% rollout with no canary analysis, and no per-provider alerting. I owned two actions: a shared HTTP client library with exponential backoff, jitter and a retry budget, adopted by all payment services within six weeks; and a 5% canary with automated rollback on the checkout SLI. Over the next two quarters our change failure rate for payments dropped from about 12% to 4%, and a similar provider degradation in March caused no customer impact because the circuit breaker failed over automatically. My lesson was to have provider failover automated and tested, not a manual flag.

Why it's strong: numbers, "I" with clear roles, mitigation first, a failed-partial mitigation handled calmly, real root cause beyond the trigger, measurable systemic improvement, honest lesson.

### Example 2 — Capacity / cascade (no deploy)

> **S:** On a Monday morning our Kubernetes-hosted API (about 40 microservices) started returning intermittent 5xx — around 3% across several unrelated services — with no deploys since Friday.
>
> **T:** I was the platform on-call and became incident commander because multiple teams were affected.
>
> **A:** Since many unrelated services failed at a low rate, I suspected a shared dependency rather than each service. Logs showed DNS lookup timeouts, so I asked one engineer to check CoreDNS while another checked node health. CoreDNS had two replicas pinned at their CPU limits; query volume had tripled since Friday. Someone traced most queries to a new analytics service resolving an external hostname in a tight loop, amplified about five times by Kubernetes' `ndots:5` search behaviour. Mitigation came in two announced steps: scale CoreDNS to six replicas and remove its CPU limit — errors fell to 0.5% — then scale the analytics service down with its owner's agreement, which brought errors to baseline. I posted updates every 20 minutes to the engineering channel and to support.
>
> **R:** Impact was about 50 minutes of low-rate errors. Postmortem actions I drove: NodeLocal DNSCache on all nodes, CoreDNS autoscaling with cluster size, a CoreDNS latency SLO with alerts, and guidance on `ndots` and DNS caching in our service template. DNS-related incidents went from three that quarter to zero in the following two. I also learned to declare a shared-dependency hypothesis early instead of letting each team debug alone.

### Example 3 — Security incident (secrets)

> **S:** At 02:10 one night GitHub secret scanning alerted that an AWS access key had been pushed to a public repository by a contractor eight minutes earlier.
>
> **T:** I was the infrastructure on-call and the first responder until the security lead joined.
>
> **A:** I deactivated the key within five minutes, before investigating, because the key had deploy permissions in production. Then I declared a security incident and paged security. CloudTrail across all regions showed the key had been used from an unknown IP: it ran GetCallerIdentity, created an IAM user with an access key, and launched GPU instances in two regions we don't use. I captured evidence, deactivated the attacker's key, removed the user and terminated the instances, and confirmed nothing touched S3 data by checking data events. I moved the CI pipeline to a temporary scoped key so deploys could continue.
>
> **R:** No customer data was accessed and unwanted spend was about $300. Afterwards I led the migration of all CI pipelines from static keys to OIDC roles — 34 pipelines in five weeks — enabled push protection on every repository, and added SCPs denying unused regions and IAM user creation outside our admin role. A year later we had zero long-lived CI keys. The lesson: the order matters — revoke first, investigate second, clean history last.

### Handling follow-ups

Expect drilling on any sentence. Prepare answers for:
- "Why did you roll back instead of fixing forward?"
- "How did you know it was the deploy and not the provider?"
- "What would you have done if the rollback hadn't helped?"
- "Who decided X? Did anyone disagree?"
- "What was **your** specific contribution vs the team?"
- "What would you do differently?"
- "How did you make sure the action items got done?"
- "What if it had happened at 10× traffic / with no logs?"

Use the **follow-up chain** from every Part 2–4 topic as practice: root cause → detection → first check → mitigation → permanent fix → verification → fix failed → prevention → differently → what-ifs.

### Traps

| Trap | Why it hurts | Instead |
|---|---|---|
| "We" for everything | Unclear what **you** did | "I" for your actions, "we" for team outcomes |
| Root cause = "human error" | Signals blame culture | Systemic contributing factors |
| Jumping to the fix | Suggests you debug before mitigating | Show mitigation first |
| No numbers | Sounds vague/junior | Impact %, minutes, counts |
| Hero story ("I stayed up 30 hours") | Suggests poor process | Show coordination and handover |
| Hiding mistakes | Seems dishonest | Own a mistake and what it taught you |
| Only technical | Misses senior scope | Include comms, coordination, follow-through |
| Too long | Interviewer loses the thread | 2–3 minutes; offer details |
| Invented details | Falls apart under drilling | Tell true stories; if your incidents are small, tell them well |

### Building your story bank (honestly)

- Collect **3–5 real incidents** you were part of: one change-caused, one capacity/dependency, one data/security, one where you were IC/coordinator, one where you made a mistake.
- If you lack production incidents, use **drills and labs** (Part 4) and say so honestly: "In our game day, I was IC for…" — interviewers value this preparation. **Never present someone else's incident or a famous outage as your own.**
- You **can** discuss famous outages (Part 3) as "an incident I studied": "AWS's October 2025 DynamoDB outage shows why…" — good for "what's an outage you learned from?"

---

## Failure modes

| Failure | Fix |
|---|---|
| Can't remember numbers | Write your stories down now, with numbers, while you can still check |
| Story only covers the fix | Add triage, communication and follow-through |
| Freezing on follow-ups | Rehearse with a friend who keeps asking "why?" |

---

## Where you'll use it

- Behavioural rounds, hiring-manager rounds, on-call scenario rounds.
- Promotion packets and incident reviews at work use the same structure.

---

## Quick recap

- STAR: Situation (context + numbers), Task (your role), Action (half the time: mitigate first, coordinate, communicate, root-cause), Result (numbers + systemic fixes + lesson).
- Say "I" for your actions; blameless, systemic language; honest mistakes.
- Prepare 3–5 true stories and rehearse the follow-up chain.

### Interview questions

1. **Q: Tell me about an incident you handled.**
   A: Use your best STAR story (see the examples above), 2–3 minutes, then invite questions.

2. **Q: Tell me about an incident you caused.**
   A: Pick a real one; own it plainly, show how you raised it fast, how you mitigated, and which systemic guardrail you added so nobody can repeat it. Blameless includes yourself.

3. **Q: Tell me about a time a mitigation didn't work.**
   A: Show calm re-triage: new hypotheses, one change at a time, time-boxing, escalation — e.g. the partial recovery after rollback in Example 1.

4. **Q: What's an outage you learned the most from (not necessarily yours)?**
   A: Pick one from Part 3, summarise trigger → root cause → why it spread → what changed, and how you applied it ("after reading Cloudflare's November 2025 postmortem, I added size validation to our generated config files").
