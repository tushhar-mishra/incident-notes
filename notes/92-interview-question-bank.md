# 92 · Incident Interview Question Bank

> **Level:** ★★★ advanced
>
> **You will learn:** 64 incident and on-call questions asked in senior DevOps / SRE / platform interviews, grouped by theme, each with a **strong, concise answer** and a pointer to the topic with the full story. Use it for spaced practice: cover the answer, say yours aloud, then compare.
>
> **You need:** Parts 1–4.
>
> **Part 5 · Interview readiness** · Previous: [91 · "Tell me about an incident"](91-tell-me-about-an-incident.md) · Next: [93 · Cheat sheet](93-cheat-sheet.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**How to use it:** pick one theme per day. Answer each question aloud in under a minute: **direct answer → reason → example**. If you hesitate, reread the linked topic.

**The one thing to know.** Senior answers always connect **mitigation, root cause and prevention**, use **numbers**, and mention **trade-offs**.

> 📖 **That's the core.** The questions follow.

---

## A. Process and roles (01–12)

1. **What's an incident vs an alert?** — An alert is an automatic signal; an incident is real (or imminent) user/business impact needing coordinated response. Many alerts never become incidents. (01)
2. **How do you set severity?** — By impact: who, how many, how badly, how long, what's at risk (money, data, security). Declare high when unsure; downgrade later. Data loss/security = SEV1 by default. (02)
3. **What does the Incident Commander do?** — Coordinates and decides: impact, roles, "what changed?", mitigation choices, update cadence, resolution. Doesn't type commands. (05)
4. **Why mitigate before root-causing?** — Users are hurting now; rollback/failover takes minutes and is reversible, while root-causing can take hours. Investigation is better when calm. (07)
5. **Rollback vs fix forward?** — Default rollback; fix forward only when rollback is impossible or riskier (irreversible migration, security fix). (07, 23)
6. **First 15 minutes of a SEV1?** — Ack; open channel; impact (who/how many/since when); declare severity and roles; what changed; fastest safe mitigation; verify with SLI; communicate with next-update time. (07)
7. **How do you communicate during an incident?** — Fixed cadence; internal template (status, impact, actions, asks, next update); plain-language status page; separate exec/support updates; backup channels. (06)
8. **What makes a postmortem blameless?** — Assumes reasonable actions given information at the time; looks for system weaknesses; "human error" is a starting question. Needed for honest timelines. (11)
9. **What are the limits of 5 whys?** — Single linear chain, stops at people, inconsistent between investigators. Use with timelines and multiple contributing factors. (11)
10. **How do you make action items stick?** — Few, specific, owned, dated, ticketed, prioritised via error budget policy, reviewed weekly; prefer guardrails to "be careful". (11)
11. **MTTD vs MTTR, and why averages mislead?** — Detection vs recovery time; improve with alerts vs mitigation tools. Durations are skewed, so use median/p90, counts and SLO burn. (10)
12. **How do you run a good game day?** — Hypothesis, safe blast radius, stop button, roles, observers, timeline, postmortem, and fix whatever it reveals. (76, 90)

## B. Alerting, SLOs and on-call (13–20)

13. **SLI vs SLO vs SLA?** — Measurement vs internal target vs contract with penalties; SLA looser than SLO. (03)
14. **Why burn-rate alerts?** — They measure how fast users are hurt relative to the SLO; multi-window gives fast detection and few false alarms. (03)
15. **Symptom vs cause alerts?** — Page on symptoms users feel; cause signals (CPU, single pod) go to dashboards/tickets unless they predict imminent impact. (04)
16. **50 pages a week — what do you do?** — Measure actionability, delete/downgrade noise, group duplicates, replace cause alerts with SLO alerts, add `for:` durations and runbooks, weekly review. (04)
17. **What's a good runbook?** — Linked from the alert; meaning, impact, checks with expected output, mitigations with risks, verification, escalation; owner and last-tested date. (09)
18. **How do you know your alerting works?** — Watchdog/dead-man's switch to an external heartbeat, periodic test pages, external synthetics. (44)
19. **What's in a handoff?** — Open incidents, things to watch, noisy alerts with tickets, planned changes. (04)
20. **How do you protect on-call health?** — Limits on pages per shift, compensation, shadowing, follow-the-sun if possible, fix the system not the person. (04)

## C. Linux and host failures (21–28)

21. **`df` says full but `du` doesn't — why?** — Deleted-but-open files (`lsof +L1`), files under a mount point, reserved blocks. Truncate via `/proc/<pid>/fd/<n>` or restart the process. (13)
22. **Disk full on a DB host — what not to do?** — Don't delete WAL/data files by hand. Grow the volume, remove non-DB files, fix stuck slots/archiving. (13, 78)
23. **"Too many open files" — leak or limit?** — Leak if FD count climbs with flat traffic (CLOSE_WAIT); limit if it tracks traffic. Check `/proc/<pid>/limits`; systemd ignores limits.conf. (14)
24. **Load is high but CPU is idle?** — D-state tasks waiting on I/O/NFS count toward load; check `vmstat` `b`/`wa`. (15)
25. **Latency spikes at 40% CPU in containers?** — CFS throttling under CPU limits; check `nr_throttled`; raise/remove limit, align thread counts. (15)
26. **Host OOM vs cgroup OOM?** — Whole machine out of memory vs container hit its limit; different log lines and fixes. (16)
27. **Why can clock skew cause outages?** — TLS validity, signed API requests, Kerberos, tokens, leases and ordering depend on time; fix with chrony, monitor offset. (17)
28. **Why use monotonic clocks?** — Durations measured with wall clock can go negative on clock steps (Cloudflare 2017). (17, 58)

## D. Containers and Kubernetes (29–37)

29. **Exit code 137?** — SIGKILL: OOMKilled (if flagged) or kubelet after a failed liveness probe. (18, 19)
30. **CrashLoopBackOff — first steps?** — `describe` for exit code/events, `logs --previous`, what changed (image, config, secret), debug with `kubectl debug --copy-to`. (19)
31. **Pending pods?** — Read the scheduler message: insufficient requests, taints, affinity, PVC zone, max pods, quota. (20)
32. **ImagePullBackOff causes?** — Missing tag, auth, rate limits, registry/network, architecture. Mirror images, deploy by digest. (20)
33. **Many nodes NotReady at once?** — Look for the shared factor: zone, node group, OS update, CNI/DaemonSet, API endpoint, certificates. (21, 63, 64)
34. **What happens when the control plane is overloaded?** — Running pods continue; changes, scheduling and endpoint/DNS updates fail; regain control by cutting load and break-glass access (OpenAI 2024). (22, 65)
35. **ConfigMap change broke pods; why didn't `rollout undo` work?** — Plain ConfigMap edits aren't rollouts; use hash-suffixed ConfigMaps so config changes become revisions. (80)
36. **Intermittent DNS failures in Kubernetes?** — CoreDNS overload, `ndots:5` amplification, conntrack race; scale CoreDNS, NodeLocal DNSCache, lower ndots. (26, 82)
37. **How do you make deploys safe?** — Readiness probes, `maxUnavailable: 0`, canary with automated analysis and rollback, PDBs, deploy markers. (23)

## E. Changes and config (38–43)

38. **Why are config pushes riskier than code deploys?** — Often global and instant, less tested, generated by automation; one bad input hits every reader at once (Cloudflare 2019/2025, CrowdStrike, GCP 2025). (24)
39. **How do you design a safe config pipeline?** — Validate (schema, size, sanity), stage rollouts with health checks, readers keep last-known-good, kill switches, versioned rollback. (24)
40. **What's "fail open" vs "fail closed"?** — On error, allow (availability) vs block (safety). Choose per system: quota/rate checks often fail open; auth often closed. (24, 53)
41. **Why can a canary miss a bad change?** — It only watches certain signals (errors) or bakes too briefly; stalled requests (Canva) or delayed crashes (Azure AFD) slip through. (54, 69)
42. **How do OS/dependency updates cause outages with no deploy?** — Auto-updates outside change control (Datadog 2023, Heroku 2025); pin versions, patch via staged images. (25, 63, 64)
43. **How would you prevent Knight Capital?** — Automated, verified deploys; delete dead code; never reuse flags; output limits and kill switches. (49)

## F. Network, DNS, TLS (44–49)

44. **Timeout vs connection refused?** — Dropped/filtered (network/firewall) vs nothing listening (app/port). (28)
45. **Why is DNS slow to fail and slow to recover?** — Caching and TTLs (including negative caching); know your TTLs, lower them before changes. (26)
46. **How did Facebook's 2021 outage make DNS disappear?** — Backbone loss → DNS sites withdrew BGP announcements by design → resolvers couldn't reach them; tools and access depended on the same network. (59)
47. **What's a route leak?** — Announcing routes you shouldn't carry (e.g. peer routes to providers), pulling others' traffic through you (Cloudflare Jan 2026). (60)
48. **Certificate expired — why did automation fail?** — Usually the renewal path broke silently (ACME challenge, issuer creds) or the server didn't reload; monitor both expiry and renewal health. (27, 81)
49. **How do you check a server's cert and chain?** — `openssl s_client -connect host:443 -servername host -showcerts | openssl x509 -noout -dates`; verify full chain from strict clients. (27)

## G. Data and databases (50–55)

50. **"Too many connections" — how do you reason?** — Pods × pool + others vs `max_connections`; scale-out can worsen it; PgBouncer, pool math, HPA limits. (30, 83)
51. **A migration took the site down — how?** — Lock queues behind an `ALTER TABLE`, or migration load + peak saturating connections; use `lock_timeout`, online schema change, throttling. (31, 73)
52. **What's a query plan flip?** — Same query, new plan after statistics change → huge slowdown with no deploy (Clerk 2026); re-ANALYZE, raise stats target, alert per-query latency. (31, 72)
53. **Why can failover lose data or cause split brain?** — Async replicas may lack recent writes; partitions can create two primaries; use quorum, fencing, topology-aware rules (GitHub 2018). (32, 71)
54. **Is a replica a backup?** — No: deletes and corruption replicate. Backups + PITR + restore tests. (34, 46)
55. **How do you know backups work?** — Automated restore tests with data checks and timing vs RTO; alerts on missing valid backups. (34)

## H. Overload and cascades (56–60)

56. **What's a retry storm, and how do you stop one?** — Retries multiply load when a system is weak; shed load, disable inner retries, open breakers, then readmit gradually; prevent with backoff + jitter + budgets, one retry layer. (36)
57. **What's a metastable failure?** — A bad state that persists after the trigger is gone (retries, cold caches); needs active load reduction. (36, 68)
58. **Cache stampede fixes?** — Single-flight/coalescing with timeouts, stale-while-revalidate, jittered TTLs, warm-up, admission control. (37)
59. **How do failures cascade?** — Waiting (no timeouts), load redistribution, retries, health-check removal, shared dependencies; defences: timeouts, bulkheads, breakers, shedding, static stability. (38)
60. **Why didn't scaling up help (Canva, Kinesis)?** — The limit wasn't capacity: herds kill new tasks before they help; per-node thread costs grow with fleet size. Cut load first. (66, 69)

## I. Security, cost, dependencies (61–64)

61. **AWS key leaked publicly — first steps?** — Deactivate now, investigate CloudTrail in all regions, remove persistence/resources, rotate legit use, clean history last; move to OIDC. (42, 86)
62. **What did CircleCI 2023 teach?** — CI holds the keys to everything; stolen sessions bypass MFA's intent; rotate at the source; prefer short-lived OIDC credentials. (75)
63. **Bill 5× overnight — approach?** — Cost Explorer by service/usage type, confirm with live metrics, stop the loop, add budgets/anomaly alerts and concurrency caps. (43, 87)
64. **How do you prepare for a cloud region/AZ outage?** — Map hidden dependencies, static stability (pre-provisioned capacity), N+1 by AZ, quotas for DR, independent status/comms, practised failover. (39, 88)

---

## Quick recap

- Answer shape: direct answer → why → example (preferably a real outage from Part 3 or your own experience).
- Always connect mitigation, root cause and prevention.
- Use numbers and trade-offs.
