# Production Incidents and Outages: From First Page to Senior On-Call

> ✅ **Status: complete.** All 93 topics (Parts 1–5) written and self-reviewed, November 2026. Every real outage in Part 3 was written from the official postmortem linked in the topic. Details that are not public are marked **"not publicly disclosed"**. Sample command output is illustrative.

One learning path, in reading order. Each topic uses only ideas explained before it. When a later idea has to appear early, it is explained right there in a line or two, and a pointer like "(topic 36)" tells you where the full story is. You never need to jump ahead.

**Why this order?** Real outages are built out of a small number of **recurring patterns** (a bad config push, a retry storm, a DNS failure…). So you learn the **basics** first, then the **patterns** one by one, then you see how the patterns combined in **famous real outages**, then you **practise** them as 3 AM drills, and finally you learn to **tell the story** in an interview.

**Topic shapes:**

1. **Basics (Part 1):** ⚡ Core (10-minute version) · What it is · Why it exists · Analogy · How it works · Failure modes · Where you'll use it · Quick recap + interview questions.
2. **Recurring failures (Part 2) and drills (Part 4):** ⚡ Core · Symptoms · Analogy · How it happens · How to find it (commands, sample output, what each line tells you, what to check first/second/third) · How to fix it (immediate vs permanent) · How to verify the fix · Prevention · Follow-up questions.
3. **Real outages (Part 3):** ⚡ Core · Summary · Impact · Timeline · How it happened (with diagram) · Why it spread · How they found it · What they did to fix it · Permanent fix · In my stack · Follow-up questions · Source.

**Every scenario ends with the follow-up chain** a senior interviewer uses: root cause vs trigger → detection → first check → stop the impact → permanent fix → verification → "what if the fix failed?" → prevention → what you'd do differently → "what if…" variations.

**Related notes:** [Linux](../linux-mastery-rhel/INDEX.md) · [Docker](../Docker/INDEX.md) · [Kubernetes](../CKA-Certification-Course-2025/INDEX.md) · [Jenkins](../Jenkins/INDEX.md) · [GitHub Actions](../GitHub-Actions-Basics-To-Production/notes/INDEX.md) · [Terraform](../Terraform/INDEX.md) · [Ansible](../Ansible/INDEX.md) · [System Design](../System-Design/INDEX.md). Topics link straight to the matching topic file wherever an incident builds on them.

**Levels:** ★ foundation · ★★ intermediate · ★★★ advanced (senior).

## How to study with these notes

| Your goal | Read | Time |
|---|---|---|
| **First pass: get the whole picture** | Only the **⚡ Core** of every topic, in order | ~15 hours |
| **Become on-call ready** | Full Parts 1–2, then do every drill in Part 4 in a lab | A few weeks |
| **Interview tomorrow** | [07](notes/07-the-first-15-minutes.md), [11](notes/11-blameless-postmortems.md), [12](notes/12-the-root-cause-map.md), any five outages from Part 3, then [91](notes/91-tell-me-about-an-incident.md), [92](notes/92-interview-question-bank.md), [93](notes/93-cheat-sheet.md) | An evening |
| **Production is on fire right now** | [07](notes/07-the-first-15-minutes.md) → [93](notes/93-cheat-sheet.md) → the Part 2 topic that matches your symptom | Minutes |

**How to use a scenario:** read only the **Symptoms** first. Stop. Write down what you would check first, second and third, and what you think the cause is. Then read on and compare.

---

## Part 1 · Incident basics

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 01 | ★ | [What is an incident](notes/01-what-is-an-incident.md) | Incident vs alert vs bug vs problem, the incident lifecycle from detect to learn |
| 02 | ★ | [Severity levels and impact](notes/02-severity-levels-and-impact.md) | SEV1–SEV4, measuring impact, when to declare, upgrading and downgrading |
| 03 | ★ | [SLOs and error budgets for on-call](notes/03-slos-and-error-budgets-for-on-call.md) | SLI, SLO, SLA and error budgets, and why they decide what wakes you up |
| 04 | ★ | [On-call and alerting](notes/04-on-call-and-alerting.md) | Rotations, paging, symptom vs cause alerts, alert fatigue, handoffs |
| 05 | ★ | [Incident roles and the Incident Commander](notes/05-incident-roles-and-commander.md) | IC, Ops lead, Comms lead, Scribe, and why the IC does not type commands |
| 06 | ★ | [Incident communication](notes/06-incident-communication.md) | Update templates, status pages, stakeholders, cadence |
| 07 | ★★ | [The first 15 minutes](notes/07-the-first-15-minutes.md) | "What changed?", mitigate before root cause, the mitigation toolbox |
| 08 | ★★ | [Debugging under pressure](notes/08-debugging-under-pressure.md) | Golden signals, USE and RED, hypotheses, the core command kit for Linux, Docker, Kubernetes and cloud |
| 09 | ★ | [Runbooks and playbooks](notes/09-runbooks-and-playbooks.md) | What a good runbook contains, keeping it alive, automating it |
| 10 | ★★ | [Incident metrics](notes/10-incident-metrics.md) | MTTD, MTTA, MTTM, MTTR, MTBF, change failure rate, and why averages lie |
| 11 | ★★ | [Blameless postmortems](notes/11-blameless-postmortems.md) | Template, timeline, contributing factors, limits of "5 whys", action items that get done |
| 12 | ★★ | [The root-cause map](notes/12-the-root-cause-map.md) | The seven pattern families used in the rest of these notes, and how to spot yours |

## Part 2 · Recurring production failures

### A. Linux host resources

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 13 | ★ | [Disk full](notes/13-disk-full.md) | Blocks vs inodes, deleted-but-open files, log growth, `df` / `du` / `lsof` |
| 14 | ★★ | ["Too many open files"](notes/14-too-many-open-files.md) | File descriptors, ulimits, `LimitNOFILE`, finding descriptor leaks |
| 15 | ★★ | [High CPU, load and throttling](notes/15-high-cpu-load-and-throttling.md) | Load average vs CPU, steal time, CFS throttling in containers |
| 16 | ★★ | [Memory leaks and the OOM killer](notes/16-memory-leaks-and-oom-killer.md) | RSS, page cache, how the OOM killer picks a victim, spotting a leak |
| 17 | ★★ | [Clock skew and NTP](notes/17-clock-skew-and-ntp.md) | chrony, why skew breaks TLS, tokens and databases, leap seconds |

### B. Containers and Kubernetes

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 18 | ★ | [OOMKilled in Kubernetes](notes/18-oomkilled-in-kubernetes.md) | Requests vs limits, exit code 137, QoS classes |
| 19 | ★ | [CrashLoopBackOff](notes/19-crashloopbackoff.md) | Backoff timing, `--previous` logs, the usual causes in order |
| 20 | ★ | [Pods Pending and ImagePullBackOff](notes/20-pods-pending-and-imagepullbackoff.md) | Scheduling failures, registry auth and rate limits, tags vs digests |
| 21 | ★★ | [Node NotReady and evictions](notes/21-node-notready-and-evictions.md) | Kubelet, pressure conditions, eviction thresholds, draining |
| 22 | ★★★ | [Control-plane overload](notes/22-control-plane-overload.md) | API server and etcd under load, LIST storms, "the cluster won't answer" |

### C. Changes

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 23 | ★ | [Bad deploys](notes/23-bad-deploys.md) | Rollback first, canaries, health gates, "it passed in staging" |
| 24 | ★★ | [Bad config pushes](notes/24-bad-config-pushes.md) | Global config, feature flags, staged config rollout, "fail small" |
| 25 | ★★ | [Dependency and OS updates](notes/25-dependency-and-os-updates.md) | Unpinned versions, unattended upgrades, base-image drift |

### D. Network, DNS, certificates

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 26 | ★★ | [DNS failures](notes/26-dns-failures.md) | Resolution chain, TTL, negative caching, CoreDNS and `ndots` |
| 27 | ★ | [Expired TLS certificates](notes/27-expired-tls-certificates.md) | Chains, intermediates, kubeadm certificates, cert-manager, expiry monitoring |
| 28 | ★★ | [Network and firewall changes](notes/28-network-and-firewall-changes.md) | Security groups, NetworkPolicy, routes, BGP in one page |
| 29 | ★★ | [Load balancer and health-check failures](notes/29-load-balancer-and-health-checks.md) | All targets unhealthy, deep vs shallow checks, draining |

### E. Data

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 30 | ★★ | [Connection-pool exhaustion](notes/30-connection-pool-exhaustion.md) | Pool math, `max_connections`, leaks, PgBouncer |
| 31 | ★★★ | [Slow queries, locks and migrations](notes/31-slow-queries-locks-and-migrations.md) | Lock queues, query plan flips, online schema change |
| 32 | ★★★ | [Replication lag, failover and split brain](notes/32-replication-failover-split-brain.md) | Fencing, quorum, promoting the wrong replica |
| 33 | ★★ | [ID and integer overflow](notes/33-id-and-integer-overflow.md) | 32-bit keys, sequences running out, seeing it coming |
| 34 | ★★ | [Backups that don't restore](notes/34-backups-that-dont-restore.md) | Untested backups, RPO and RTO, restore drills |

### F. Overload and cascades

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 35 | ★★ | [Quotas and rate limits](notes/35-quotas-and-rate-limits.md) | Cloud API throttling, service quotas, 429s, your own limits |
| 36 | ★★★ | [Retry storms](notes/36-retry-storms.md) | Retry amplification math, backoff + jitter, retry budgets |
| 37 | ★★★ | [Cache stampede and thundering herd](notes/37-cache-stampede-thundering-herd.md) | Cold caches, request coalescing, staggered TTLs |
| 38 | ★★★ | [Cascading failures](notes/38-cascading-failures.md) | Feedback loops, circuit breakers, load shedding, metastable failures |
| 39 | ★★ | [Third-party and cloud-region outages](notes/39-third-party-and-region-outages.md) | Hidden dependencies, control planes vs data planes, static stability |

### G. Human error, security, cost

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 40 | ★ | [`rm -rf` and the wrong environment](notes/40-rm-rf-and-wrong-environment.md) | Prompts, guardrails, least privilege, the two-person rule |
| 41 | ★★ | [Terraform destroying production](notes/41-terraform-destroying-production.md) | Replace plans, `prevent_destroy`, state mistakes, plan review |
| 42 | ★★ | [Secrets leaked in Git](notes/42-secrets-leaked-in-git.md) | Rotate first, then clean history; secret scanning; OIDC instead of static keys |
| 43 | ★★ | [Runaway cloud costs](notes/43-runaway-cloud-costs.md) | Loops, NAT and egress, log bills, budgets and anomaly alerts |
| 44 | ★★ | [When monitoring goes dark](notes/44-when-monitoring-goes-dark.md) | Observability that depends on the thing that broke, out-of-band checks |

## Part 3 · Famous real outages, grouped by pattern

| # | Level | Outage | Pattern you'll recognise |
|---|---|---|---|
| | | **Human error and tooling** | |
| 45 | ★ | [AWS S3, February 2017](notes/45-aws-s3-2017.md) | A typo in a playbook command + a huge blast radius |
| 46 | ★ | [GitLab database deletion, January 2017](notes/46-gitlab-2017.md) | Wiped the database directory on the wrong server + backups that didn't work |
| 47 | ★★ | [Atlassian, April 2022](notes/47-atlassian-2022.md) | A cleanup script given the wrong IDs, slow restore |
| 48 | ★★ | [Google Cloud × UniSuper, May 2024](notes/48-google-cloud-unisuper-2024.md) | A blank parameter in an internal tool deleted a whole private cloud |
| | | **Bad deploys and config pushes** | |
| 49 | ★ | [Knight Capital, August 2012](notes/49-knight-capital-2012.md) | A partial deploy + a reused feature flag |
| 50 | ★★ | [Cloudflare regex, July 2019](notes/50-cloudflare-regex-2019.md) | A global config push that used 100% CPU |
| 51 | ★★ | [Fastly, June 2021](notes/51-fastly-2021.md) | A latent bug triggered by one customer's valid config |
| 52 | ★★ | [CrowdStrike, July 2024](notes/52-crowdstrike-2024.md) | A content update pushed to every machine at once |
| 53 | ★★★ | [Google Cloud Service Control, June 2025](notes/53-google-cloud-service-control-2025.md) | Unprotected code path, no feature flag, no backoff |
| 54 | ★★★ | [Azure Front Door, October 2025](notes/54-azure-front-door-2025.md) | Valid customer configs → incompatible metadata → async crash that passed health gates and poisoned the "last known good" |
| 55 | ★★★ | [Cloudflare, 18 November 2025](notes/55-cloudflare-nov-2025.md) | A database permission change → oversized feature file → crash |
| 56 | ★★ | [Cloudflare, 5 December 2025](notes/56-cloudflare-dec-2025.md) | A config change exposed a years-old bug |
| | | **Certificates and time** | |
| 57 | ★ | [Certificate expiry outages](notes/57-certificate-expiry-outages.md) | Ericsson 2018 (expired certificate in mobile core software) and the Let's Encrypt root expiry 2021 |
| 58 | ★★ | [Cloudflare leap second, January 2017](notes/58-cloudflare-leap-second-2017.md) | Time went "backwards" and code assumed it couldn't |
| | | **DNS and network** | |
| 59 | ★★ | [Facebook BGP, October 2021](notes/59-facebook-bgp-2021.md) | A network command cut off the backbone, and the tools to fix it |
| 60 | ★★ | [Cloudflare route leak, January 2026](notes/60-cloudflare-route-leak-2026.md) | An automated BGP policy change became too permissive and leaked routes |
| 61 | ★★ | [Cloudflare 1.1.1.1, July 2025](notes/61-cloudflare-1111-2025.md) | A dormant config error activated weeks later |
| 62 | ★★★ | [AWS us-east-1 DynamoDB DNS, October 2025](notes/62-aws-dynamodb-dns-2025.md) | A race condition in DNS automation → an empty DNS record |
| | | **Platform, OS and Kubernetes** | |
| 63 | ★★ | [Datadog, March 2023](notes/63-datadog-2023.md) | A legacy security-update channel updated systemd on many nodes at once; restarting networkd removed CNI routing rules |
| 64 | ★★ | [Heroku, June 2025](notes/64-heroku-2025.md) | An automated OS upgrade restarted networking; routes not reapplied; tools and status page on the same infrastructure |
| 65 | ★★★ | [OpenAI, December 2024](notes/65-openai-kubernetes-2024.md) | A telemetry service overloaded the Kubernetes API → DNS failed |
| | | **Capacity and overload** | |
| 66 | ★★ | [AWS Kinesis, November 2020](notes/66-aws-kinesis-2020.md) | Adding capacity hit an OS thread limit |
| 67 | ★★ | [Slack, January 2021](notes/67-slack-2021.md) | A traffic spike + cloud network scaling + autoscaling feedback |
| 68 | ★★★ | [AWS us-east-1, December 2021](notes/68-aws-us-east-1-2021.md) | Automated scaling → internal network congestion → retry storm |
| 69 | ★★★ | [Canva, November 2024](notes/69-canva-2024.md) | A thundering herd of 1.5 million requests per second |
| 70 | ★★★ | [Roblox, October 2021](notes/70-roblox-2021.md) | Consul streaming + BoltDB, 73 hours, circular dependencies |
| | | **Data and databases** | |
| 71 | ★★★ | [GitHub, October 2018](notes/71-github-2018.md) | A 43-second network partition → cross-country database failover |
| 72 | ★★ | [Clerk, February 2026](notes/72-clerk-2026.md) | Postgres auto-ANALYZE → query plan flip |
| 73 | ★★ | [GitHub 2026 availability reports](notes/73-github-2026-mini-cases.md) | Six mini-cases: mass cache expiry, migration saturating DB connections, 32-bit key overflow, automation suspending a service account, a deploy exhausting headroom, timeouts → OOM |
| | | **Automation and security** | |
| 74 | ★★ | [Cloudflare BYOIP, February 2026](notes/74-cloudflare-byoip-2026.md) | Cleanup automation deleted customer IP prefixes |
| 75 | ★★ | [CircleCI security incident, January 2023](notes/75-circleci-2023.md) | A stolen session → rotate every secret |

## Part 4 · On-call drills: "You get this alert at 3 AM"

| # | Level | Drill | What you'll practise |
|---|---|---|---|
| 76 | ★ | [How to run drills and game days](notes/76-how-to-run-drills.md) | Breaking things safely in a lab, scoring yourself |
| 77 | ★ | [5xx spike right after a deploy](notes/77-drill-5xx-after-deploy.md) | Correlating with changes, rolling back first |
| 78 | ★ | [Disk at 95% on the database host](notes/78-drill-disk-95-database.md) | Safe space recovery on a database server |
| 79 | ★★ | [p99 latency up, nothing deployed](notes/79-drill-latency-no-deploy.md) | Finding the change that isn't a deploy |
| 80 | ★★ | [CrashLoopBackOff after a ConfigMap change](notes/80-drill-crashloop-after-configmap.md) | Config as a change, rollback of config |
| 81 | ★★ | ["certificate has expired" on an internal service](notes/81-drill-certificate-expired.md) | Finding and renewing the right certificate |
| 82 | ★★ | [Intermittent DNS failures in the cluster](notes/82-drill-dns-intermittent.md) | CoreDNS, conntrack, `ndots`, NodeLocal DNSCache |
| 83 | ★★ | ["Too many connections" from the database](notes/83-drill-too-many-connections.md) | Pool math across many pods |
| 84 | ★★★ | [Queue backlog growing, retries climbing](notes/84-drill-queue-backlog-retries.md) | Stopping a retry storm, draining a backlog |
| 85 | ★★ | [Terraform plan wants to destroy the prod database](notes/85-drill-terraform-destroy-prod-db.md) | Stopping the pipeline, reading the plan, safe fix |
| 86 | ★★ | [AWS key pushed to a public repo](notes/86-drill-aws-key-leaked.md) | Revoke, investigate CloudTrail, clean up |
| 87 | ★★ | [Cloud bill 5× overnight](notes/87-drill-cloud-bill-5x.md) | Cost Explorer, finding the runaway, stopping it |
| 88 | ★★★ | [One Availability Zone is gone](notes/88-drill-az-down.md) | Evacuating an AZ, static stability |
| 89 | ★★★ | [No logs, no dashboards: monitoring is down too](notes/89-drill-monitoring-down.md) | Debugging blind, out-of-band tools |
| 90 | ★★★ | [Capstone: a game day with several faults](notes/90-drill-game-day-capstone.md) | Running a full incident with roles and a postmortem |

## Part 5 · Interview readiness

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 91 | ★★ | ["Tell me about a production incident you handled"](notes/91-tell-me-about-an-incident.md) | The STAR method, senior-level example answers, traps to avoid |
| 92 | ★★★ | [Incident interview question bank](notes/92-interview-question-bank.md) | 60+ questions with strong answers, by theme |
| 93 | ★ | [Cheat sheet](notes/93-cheat-sheet.md) | Pattern recognition table + the first commands to run for each symptom |
