# 93 · Cheat Sheet

> **Level:** ★ foundation
>
> **You will learn:** nothing new — this is the **one page to open at 3 AM**: the first-15-minutes script, symptom → likely family → first commands, the mitigation toolbox, communication templates, and the follow-up chain for interviews.
>
> **You need:** Parts 1–4.
>
> **Part 5 · Interview readiness** · Previous: [92 · Interview question bank](92-interview-question-bank.md) · [Back to INDEX](../INDEX.md)

---

## 1. The first 15 minutes (topic 07)

```
0–2   ACK · open #inc channel · "looking"
2–5   IMPACT: what users see · how many · since when (exact minute) · getting worse?  → SEV
5–8   WHAT CHANGED: deploys · config/flags · infra (TF/Ansible) · OS/agents · traffic · deps · certs/time · jobs
8–12  MITIGATE (fastest safe, ONE at a time, announced): rollback · flag off · shift traffic · failover · scale · shed · restart
12–15 VERIFY with the user-facing SLI · post update with NEXT UPDATE TIME · upgrade SEV if needed
```

## 2. Symptom → likely family → first commands

| Symptom | Likely family (topic) | First commands |
|---|---|---|
| Step up in errors right after a deploy | Bad deploy (23, 77) | `kubectl rollout history deploy/X`; compare old/new pods; **roll back** |
| Everything fails at once, no deploy | Config push / shared dep / DNS (24, 26, 39) | config & flag audit logs; `dig`; provider status |
| Errors start at an exact time (00:00) | Cert / time / cron (17, 27) | `openssl s_client ... \| openssl x509 -noout -dates`; `chronyc tracking`; `kubectl get cronjobs -A` |
| Latency ramp, nothing deployed | Saturation, batch job, plan flip, dependency (79) | per-hop latency/traces; `pg_stat_activity`; pool waiters; `kubectl top` |
| Many services, small % errors, DNS timeouts | CoreDNS / resolver (26, 82) | `kubectl -n kube-system top pods -l k8s-app=kube-dns`; CoreDNS logs |
| `No space left on device` | Disk (13, 78) | `df -h; df -i`; `du -xh --max-depth=1 /var \| sort -h`; `lsof +L1` |
| `Too many open files` | FDs (14) | `ls /proc/$PID/fd \| wc -l`; `grep "open files" /proc/$PID/limits`; `ss -tanp` |
| Restarts, exit 137, `OOMKilled` | Memory (16, 18) | `kubectl describe pod` (Last State); `dmesg -T \| grep -i oom` |
| `CrashLoopBackOff` | Config/deps/command/probe (19, 80) | `kubectl logs --previous`; `describe` events; what changed |
| `Pending` / `ImagePullBackOff` | Scheduling / registry (20) | `kubectl describe pod` → Events |
| Nodes NotReady (many) | Shared node issue (21, 63, 64) | `kubectl get nodes -L zone,nodegroup`; kubelet/containerd logs; package history |
| kubectl times out, deploys fail | Control-plane overload (22, 65) | API server metrics by client/verb; etcd status; break-glass |
| `too many clients` / pool timeouts | Connections (30, 83) | `pg_stat_activity` by client/state; pods × pool vs `max_connections` |
| Requests hang behind a migration | Locks (31) | `pg_blocking_pids`; cancel the waiting DDL |
| Load to a dependency ≫ user traffic | Retry storm (36, 84) | retries/ratio metrics; mesh retry config; shed load |
| Hit ratio collapse, DB spike | Stampede/herd (37, 69) | cache stats/uptime; top queries; readmit gradually |
| Timeout vs refused vs reset | Network (28) | `nc -zv host port`; `curl -v`; `tcpdump`; SG/NetworkPolicy |
| LB 503 with healthy app | Health checks (29) | target health reasons; `curl` the health path |
| Inserts fail, reads fine, "out of range" | ID overflow (33) | `pg_sequences` pct used; column types |
| `Signature expired`, x509 on some hosts | Clock skew (17) | `timedatectl`; `chronyc tracking` |
| Quota/throttling errors on scale-up | Quotas (35) | read the error; Service Quotas usage |
| Spend spike | Runaway cost (43, 87) | Cost Explorer by service/usage type; live usage metrics |
| Secret in a public repo | Leak (42, 86) | **deactivate key**; CloudTrail all regions |
| Dashboards empty, no alerts, users complain | Monitoring dark (44, 89) | `curl` from outside; `kubectl get/top/logs`; cloud metrics |
| One zone's nodes/targets all down | AZ outage (88) | `kubectl get nodes -L topology.kubernetes.io/zone`; provider health |

## 3. Mitigation toolbox (topic 07)

| Action | Command / lever |
|---|---|
| Roll back (K8s) | `kubectl rollout undo deploy/X` (GitOps: `git revert` + sync) |
| Roll back (Helm) | `helm rollback REL REV` |
| Feature off | flag tool kill switch |
| Shift traffic | LB/DNS weights, `kubectl cordon`, mesh routing |
| Fail over | DB promote (managed: failover API), regional DNS |
| Scale | `kubectl scale`, HPA max, node group size (watch quotas) |
| Shed load | WAF/rate limits, disable non-critical features, pause batch |
| Restart | `kubectl rollout restart deploy/X` (grab evidence first) |
| Stop automation | pause pipelines/CronJobs/generators; kill switches |

**Rules:** one change at a time · announce before acting · time-box · keep evidence · verify with user-facing SLI.

## 4. Communication templates (topic 06)

```
[SEV1] <title> — update #N at HH:MM UTC
Status: Investigating | Identified | Mitigating | Monitoring | Resolved
Impact: <who/how many/what> since HH:MM
Cause: <suspected/confirmed>
Actions: <now> (ETA if known)
Asks: <deploy freeze / support macro>
IC / Ops / Comms: …        Next update: HH:MM UTC
```
Status page: plain language · what still works · next update time · no internal names · no blame.

## 5. Postmortem skeleton (topic 11)

Summary · Impact (numbers, budget) · Timeline (UTC) · Contributing factors (trigger + latent weakness + failed defences) · Went well / badly / lucky · Actions (Prevent / Detect / Mitigate — owner, date, ticket).

## 6. Seven pattern families (topic 12)

Change · Human error/tooling · Capacity/overload · Dependency · DNS/network · Certificates/time · Automation gone wrong. Big outages = **trigger + spreader + amplifier**.

## 7. Follow-up chain (every interview scenario)

1. Root cause vs trigger 2. Detection (why not earlier) 3. First check (why) 4. Stop impact 5. Permanent fix 6. Verification 7. If the fix failed 8. Prevention 9. Differently next time 10. What-ifs (10× traffic, during deploy, no logs, no access).

## 8. Real outages in one line each (Part 3)

| # | Outage | One-line lesson |
|---|---|---|
| 45 | AWS S3 2017 | Tools must limit how fast/how much they can remove |
| 46 | GitLab 2017 | Untested backups aren't backups |
| 47 | Atlassian 2022 | Destructive APIs must reject ambiguous IDs; restore at the scale you delete |
| 48 | GCP × UniSuper 2024 | Blank must not mean a dangerous default; keep independent backups |
| 49 | Knight 2012 | Verify every server got the deploy; never reuse flags; kill switches |
| 50 | Cloudflare 2019 | Regex/rules are code: perf tests + staged rollout |
| 51 | Fastly 2021 | Latent bugs + tenant config; isolation and fast disable |
| 52 | CrowdStrike 2024 | "Just content" needs rings; bounds-check inputs |
| 53 | GCP 2025 | Flags off by default, fail open, randomized backoff |
| 54 | Azure AFD 2025 | Health gates shorter than time-to-failure; don't poison "last known good" |
| 55 | Cloudflare Nov 2025 | Validate generated config like user input |
| 56 | Cloudflare Dec 2025 | Every path to prod needs staging, including kill switches |
| 57 | Cert expiries | Inventory embedded certs; test old clients before root changes |
| 58 | Leap second 2017 | Monotonic clocks for durations |
| 59 | Facebook 2021 | Recovery tools must not depend on what failed |
| 60 | Route leak 2026 | Removing a filter can widen a rule |
| 61 | 1.1.1.1 2025 | Dormant config + global refresh; stage topology changes |
| 62 | AWS 2025 | Automation needs freshness checks; recovery is a load event |
| 63 | Datadog 2023 | Control every automatic update channel |
| 64 | Heroku 2025 | Boot-only config is a time bomb; tools off the platform |
| 65 | OpenAI 2024 | Per-node agents multiply cost; break-glass API access |
| 66 | Kinesis 2020 | Per-node resources that grow with fleet size |
| 67 | Slack 2021 | Autoscale on the right signal; test provisioning at scale |
| 68 | AWS 2021 | Backoff is a safety feature |
| 69 | Canva 2024 | Waiting clients are future load; recover by gradual admission |
| 70 | Roblox 2021 | Shared infra SPOFs; monitoring must be independent |
| 71 | GitHub 2018 | Topology-aware failover; integrity over speed |
| 72 | Clerk 2026 | Databases change without deploys; watch plans |
| 73 | GitHub 2026 | Same patterns, new combinations; headroom matters |
| 74 | Cloudflare BYOIP 2026 | Empty must never mean "all" |
| 75 | CircleCI 2023 | CI is a crown jewel; OIDC; rotate at the source |
