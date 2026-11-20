# 12 · The Root-Cause Map

> **Level:** ★★ intermediate
>
> **You will learn:** the **seven pattern families** that almost every production incident falls into (**change, human error, capacity/overload, dependency, DNS/network, certificates/time, automation gone wrong**), the **fingerprint** of each (what the graphs and errors look like), the **first question** to ask for each, how patterns **combine** in big outages, and a map of every topic in Parts 2 and 3 by family.
>
> **You need:** topics 01–11.
>
> **Part 1 · Incident basics** · Previous: [11 · Blameless postmortems](11-blameless-postmortems.md) · Next: [13 · Disk full](13-disk-full.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Thousands of different incidents come from a **small number of patterns**. Experienced on-call engineers aren't faster because they know every system; they're faster because they **recognise the pattern** in the first few minutes.

| # | Family | Fingerprint | First question |
|---|---|---|---|
| 1 | **Change** (deploy, config, flag, upgrade) | Sudden **step** in errors right after a change | "What changed just before it started?" |
| 2 | **Human error / tooling** | A manual action; wrong target, wrong input | "Who ran what, where?" (audit log) |
| 3 | **Capacity / overload** | **Ramp** or spike in traffic, saturation, latency first then errors | "Is demand up or capacity down?" |
| 4 | **Dependency** | Our errors match another service's problem | "Which dependency is slow or failing?" |
| 5 | **DNS / network** | Many unrelated services fail at once; "can't resolve", timeouts | "Can things find and reach each other?" |
| 6 | **Certificates / time** | Sudden failure at an exact time, often a round hour; TLS errors | "Did something expire or did the clock jump?" |
| 7 | **Automation gone wrong** | Machines doing damage fast, often at scale | "What automated process acted, and on what input?" |

**Picture it.** A car mechanic hears a noise and says "that's the brakes" in two seconds. They've heard it a hundred times. This topic gives you the hundred noises.

**The one thing to know.** Big outages are usually **two or three patterns combined**: e.g. a **change** triggers a failure, **automation** spreads it everywhere, and a **retry storm** (overload) stops it from recovering.

**Try it now:** for the last incident you saw, which family was the trigger, and which family made it worse?

**You've got it if you can:** look at a graph shape and an error message and name the likely family.

> 📖 **That's the core.** Everything below is the **full reference** and the map of the rest of the notes.

---

## What it is

A **pattern family** is a group of incidents with the same kind of root cause. Knowing the family tells you:
- **Where to look first.**
- **Which mitigation** usually works (rollback for changes, shed load for overload, failover for dependencies).
- **Which prevention** fixes the whole class (staged rollout for changes, guardrails for human error).

---

## Why it exists

You can't memorise every system's failure modes. You **can** learn seven families well. Then a new system's incident looks familiar: "this is a retry storm, just in a different service".

---

## Analogy

**A doctor's differential diagnosis.** Chest pain? Heart, lungs, muscles, stomach. The doctor checks the most likely and most dangerous first. Seven families are your list of "organs" to check.

---

## How it works

### The seven families in detail

**1. Change**
- Includes: code deploys, config pushes, feature flags, schema migrations, infrastructure applies, platform upgrades (Kubernetes, OS), dependency upgrades.
- Fingerprint: step change; often limited at first to new pods/nodes/regions that received the change.
- Mitigation: **roll back**. Prevention: staged rollouts, canaries, validation, feature flags, "fail small".
- Industry data: change is consistently the most common trigger of outages. (Google's SRE book states that about 70% of outages at Google are due to changes in a live system.)

**2. Human error / tooling**
- Includes: wrong command, wrong environment, wrong input, deleted the wrong thing.
- Fingerprint: audit log shows a manual action right before; often large and sudden.
- Mitigation: stop the action, restore. Prevention: **make the tool safe** (limits, confirmations, dry-runs, least privilege), not "be careful".

**3. Capacity / overload**
- Includes: traffic spikes, slow resource exhaustion (disk, memory, file descriptors, connections, IDs), retry storms, cache stampedes, cascading failures.
- Fingerprint: saturation metrics rise first; latency rises before errors; often time-of-day related.
- Mitigation: shed load, scale, rate limit, break the feedback loop. Prevention: headroom, limits, backoff, load tests.

**4. Dependency**
- Includes: cloud provider outage, third-party API, internal shared service, database.
- Fingerprint: your errors line up exactly with one dependency's latency/errors; timeouts in logs.
- Mitigation: fail over, degrade gracefully, circuit-break. Prevention: timeouts, fallbacks, multi-provider or multi-region, knowing your hidden dependencies.

**5. DNS / network**
- Includes: DNS records wrong or missing, resolvers overloaded, BGP withdrawals, firewall/security group changes, routes removed, MTU, partitions.
- Fingerprint: many services fail at once; errors like `NXDOMAIN`, `SERVFAIL`, `no such host`, `connection timed out`, `no route to host`.
- Mitigation: revert the change; restore records; failover. Prevention: staged network changes, DNS caching with sane TTLs, out-of-band access.

**6. Certificates / time**
- Includes: expired TLS certificates, expired tokens or keys, clock skew, leap seconds, daylight saving, year boundaries, date bugs.
- Fingerprint: sudden failure at a specific time (often 00:00 UTC or a round hour); `certificate has expired`, `x509`, signature/token errors.
- Mitigation: renew, fix clock. Prevention: automated renewal + expiry monitoring; time-safe code.

**7. Automation gone wrong**
- Includes: scripts deleting the wrong things, autoscalers fighting each other, control loops acting on bad data, self-healing that makes it worse.
- Fingerprint: damage at machine speed and scale; often a "cleanup" or "sync" job.
- Mitigation: **stop the automation** (kill switch), then restore. Prevention: blast-radius limits, rate limits, sanity checks on inputs ("refuse to delete more than 5%"), dry-runs.

### How patterns combine

```
Trigger            Spreader                  Amplifier                 Result
CHANGE        →    AUTOMATION pushes it   →  OVERLOAD (retries,     →  global outage
(bad config)       to every server in         restart storms)           that's slow to
                   seconds                                              recover
```

Examples from Part 3:
- **Cloudflare 2019** (topic 50): change (regex rule) + global push + CPU exhaustion.
- **AWS 2025** (topic 62): automation (DNS management race) + DNS (empty record) + overload during recovery.
- **OpenAI 2024** (topic 65): change (new telemetry service) + overload (Kubernetes API) + DNS dependency.
- **Roblox 2021** (topic 70): capacity/software issue in a dependency (Consul) + circular dependencies that slowed recovery.

### The map of this course by family

| Family | Part 2 patterns | Part 3 real outages |
|---|---|---|
| **Change** | 23 bad deploys · 24 bad config pushes · 25 dependency & OS updates · 31 migrations | 49 Knight · 50 Cloudflare 2019 · 51 Fastly · 52 CrowdStrike · 53 GCP 2025 · 54 Azure AFD · 55, 56 Cloudflare 2025 · 63 Datadog · 64 Heroku |
| **Human error / tooling** | 40 rm -rf & wrong env · 41 Terraform destroy · 34 backups that don't restore (what saves you) | 45 AWS S3 · 46 GitLab · 47 Atlassian · 48 GCP/UniSuper |
| **Capacity / overload** | 13–16 host resources · 18 OOMKilled · 22 control plane · 30 pools · 33 IDs · 35 quotas · 36 retry storms · 37 stampede · 38 cascades | 65 OpenAI · 66 Kinesis · 67 Slack · 68 AWS 2021 · 69 Canva · 70 Roblox · 72 Clerk · 73 GitHub 2026 |
| **Dependency** | 39 third-party & region outages | 61 Cloudflare 1.1.1.1 (its users) · 62 AWS 2025 (everyone on us-east-1) · 67 Slack · 70 Roblox |
| **DNS / network** | 26 DNS · 28 network & firewall · 29 load balancers · 32 split brain | 59 Facebook · 60 Cloudflare route leak · 61 Cloudflare 1.1.1.1 · 62 AWS 2025 · 71 GitHub 2018 |
| **Certificates / time** | 17 clock skew · 27 TLS certs | 57 certificate expiry · 58 leap second |
| **Automation gone wrong** | 24 config pushes · 21 evictions | 62 AWS 2025 · 68 AWS 2021 · 74 Cloudflare BYOIP |
| **Security** (often its own process) | 42 secrets in Git | 75 CircleCI |
| **Cost** (slow incidents) | 43 runaway costs | — |
| **Observability** (makes everything worse) | 44 monitoring goes dark | 59 Facebook · 65 OpenAI |

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Pattern-matching too fast** | "It's DNS" — but it wasn't | Treat the family as a **first hypothesis**, then test |
| **Stopping at the trigger** | "It was a bad config push" | Also name the spreader and the amplifier |
| **Only one family per incident** | Fix one; the next incident hits the other | List every family involved in the postmortem |

---

## Where you'll use it

- Everything after this topic is organised by these families.
- **Cheat sheet** (topic 93) turns this map into a one-page symptom → family → first command table.

---

## Quick recap

- Seven families: change, human error, capacity, dependency, DNS/network, certificates/time, automation.
- Each has a fingerprint (graph shape + error messages) and a first question.
- Big outages = trigger + spreader + amplifier.
- Use the family as your first hypothesis, then test it.

### Interview questions

1. **Q: What are the most common causes of production outages?**
   A: Changes are the most common trigger: deploys, config pushes, flags, upgrades and infrastructure applies. Then capacity and overload (including retry storms and resource exhaustion), dependency failures, DNS/network problems, certificate or time issues, human error with tools, and automation acting on bad input. Big outages usually combine a trigger with something that spreads it and something that amplifies it.

2. **Q: Many unrelated services all started failing at the same moment. What do you suspect first?**
   A: Something they all share: DNS, the network, a shared dependency (auth, database, config service), a certificate, or the platform (Kubernetes control plane, cloud region). I'd check name resolution and connectivity first, then the shared services' health and recent changes to them.

3. **Q: Errors appeared at exactly 00:00 UTC with no deploy. What families come to mind?**
   A: Certificates/time (an expiry, a date-dependent bug, a token/key rotation) and scheduled automation (a cron or batch job at midnight). I'd check certificate expiry, scheduled jobs, and any time-based logic.
