# 73 · GitHub 2026 Availability Reports: Six Mini-Cases

> **Level:** ★★ intermediate
>
> **Pattern:** Six short, official incident summaries from **one company in one year**, each a clean example of a Part 2 pattern: **mass cache expiry**, **migration saturating database connections**, **32-bit key overflow**, **automation suspending a critical service account**, **a routine deploy exhausting headroom**, and **waiting on timeouts until out of memory**. **Families:** change · capacity · data · automation · dependency.
>
> **You will learn:** how to read a short public incident summary like an SRE, how each case maps to a recurring failure topic, and the specific follow-ups GitHub committed to. Use this topic as **rapid pattern-recognition practice**: read each "What happened", stop, name the pattern and your first check, then read on.
>
> **You need:** Part 1, Part 2 (especially topics 18, 30, 31, 33, 36, 37, 38).
>
> **Part 3 · Real outages · Data and databases** · Previous: [72 · Clerk 2026](72-clerk-2026.md) · Next: [74 · Cloudflare BYOIP 2026](74-cloudflare-byoip-2026.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** GitHub publishes a monthly **availability report** describing each incident that degraded its services: what happened, impact, cause and follow-ups. These short write-ups are perfect for practising pattern recognition.

| # | Date (2026) | What happened (one line) | Pattern (topic) |
|---|---|---|---|
| A | 3 Mar | A deploy meant to reduce cache writes made **every user's cache expire, recalculate and rewrite** | Cache stampede (37) |
| B | 4 May | An online **schema migration** plus weekly-peak traffic **saturated database connections** | Migrations + pool exhaustion (30, 31) |
| C | 6 May | A **32-bit key** in a Vitess lookup table **hit its maximum**; new PR review threads failed | ID overflow (33) |
| D | 26 May | **Automated account review suspended the GitHub Actions service account** | Automation gone wrong (12) |
| E | 6 Aug | A **routine deploy briefly reduced pods**; services near capacity ran out of **headroom** → cascade | Cascading failure (38) |
| F | 5 Oct | Partial cloud networking failure → service **waited on timeouts** → **out of memory** → restart loop | Timeouts / cascade (36, 38) |

**The one thing to know.** Big companies' incidents are mostly the **same patterns** you've studied, in new combinations. Recognising them fast is the skill.

**You've got it if you can:** for each case, name the first check you would run and the permanent fix you would propose before reading GitHub's.

> 📖 **That's the core.** Everything below is the **full reference**. Note: GitHub's reports sometimes give slightly different times in the heading and the body; both are shown where they differ.

---

## Case A · 3 March 2026: every user's cache expired at once

**Summary.** While deploying a change to reduce the burden of writes from a user-settings caching mechanism, "a bug caused every user's cache to expire, get recalculated, and get rewritten." The increased load caused replication delays that cascaded to dependent services.

| Item | Detail |
|---|---|
| Time | Body: 18:46–20:09 UTC (heading: 18:59 UTC, lasting 1 h 10 min) |
| Impact | github.com request failures peaked at ~40%; ~43% of API requests failed; Git over HTTP ~6% errors (SSH unaffected); Copilot ~21%; Actions < 1% |
| Related | Same underlying cause as an incident in early February |
| Mitigation | "immediately rolling back the faulty deployment" |
| Follow-ups | A **killswitch** and improved monitoring for the caching mechanism; moving it to a **dedicated host** so future issues only affect services that rely on it |

**Pattern:** a synchronized cache refill = self-inflicted stampede (topic 37). **First check:** cache hit/miss rate and DB write/replication lag right after the deploy. **Lesson:** cache-key or expiry changes need staged rollout and a kill switch.

## Case B · 4 May 2026: migration + weekly peak saturated connections

**Summary.** "a routine online schema migration running against a large, heavily-accessed database table" had progressed for hours, but "as traffic ramped up toward the weekly peak, the combined load from the migration and normal production traffic saturated database connection capacity", causing query contention on a primary and cascading timeouts.

| Item | Detail |
|---|---|
| Time | Body: 15:34–16:40 UTC, ~1 h 6 min (heading: 15:45 UTC, 55 min) |
| Impact | Peak ~1.3% of requests 5xx (average ~0.46%); pull requests red at peak; Issues, Actions, webhooks, Git with latency/errors; other features degraded |
| Detection | "within approximately three minutes" via automated monitoring and on-call observation |
| Mitigation | Migration **paused**; dependent services recovered; ~33 min to mitigation |
| Follow-ups | Migrations on large hot tables in **low-traffic windows** with **dynamic throttling**; **automated circuit breakers** pausing migrations when latency or connection use crosses thresholds; monitoring migration pressure; review connection-pool capacity |

**Pattern:** topics 30 + 31. **First check:** what background DB work is running (migrations, backfills) and connection usage vs limit.

## Case C · 6 May 2026: 32-bit key reached its maximum

**Summary.** "This incident was caused by a 32-bit integer key reaching its maximum value in a Vitess lookup table used during pull request thread creation." The primary table had been migrated to a 64-bit key, but **the Vitess lookup table remained 32-bit**.

| Item | Detail |
|---|---|
| Time | Body: 15:12–19:02 UTC (heading: 15:25 UTC, 3 h 39 min) |
| Impact | Near 100% failure for new pull request review threads (line and file comments); existing PRs and comments unaffected |
| Mitigation | Updated lookup table definitions on all shards to 64-bit; service restored when schema changes completed globally |
| Follow-ups | Extend **monitoring of database column limits to Vitess lookup tables** for earlier detection |

**Pattern:** topic 33, and specifically "the main column was fixed, but a **secondary** place storing the same ID wasn't". **First check:** the exact insert error and the column/sequence max for **every** table in the write path.

## Case D · 26 May 2026: automation suspended the Actions service account

**Summary.** "our automated account review system incorrectly suspending the service account used by GitHub Actions to authenticate workflow runs and download actions."

| Item | Detail |
|---|---|
| Time | Body: 10:40–12:56 UTC (heading: 10:57 UTC, 2 h 21 min) |
| Impact | 10:40–12:16: all newly queued Actions runs failed to start; until 12:56, runs needing to download actions still failed; Pages, Copilot code review/coding agent and migration tools affected; some content briefly marked hidden (restored, no data lost) |
| Mitigation | Account restored 12:16; exempted from automated review 12:20; a service redeployed 12:48 to flush cached account state; recovered 12:56 |
| Follow-ups | **Allowlist of service accounts that cannot be suspended by automated systems**; better account diagnostics; reduced cache propagation delays |

**Pattern:** automation acting on a critical identity (topic 12, family 7). **First check:** auth errors from the platform's own service identity; audit log for account changes. **Lesson:** protect critical machine identities from automated enforcement.

## Case E · 6 August 2026: a routine deploy exhausted headroom

**Summary.** Actions services were "running close to their capacity and concurrency limits. A routine deployment that briefly reduced the number of running pods was enough to exhaust available headroom." This "caused service mesh sidecars to experience CPU throttling and out-of-memory restarts, which then cascaded into cache, DNS, and API errors across multiple clusters." GitHub notes "The deployment's contents were not at fault (we rolled it back to confirm this)." A latent bug in job assignment made recovery slower (runners received already-revoked jobs and got stuck retrying).

| Item | Detail |
|---|---|
| Time | Heading: 15:22 UTC, 10 h 42 min (summary also says ~9 hours) |
| Impact | Actions (hosted and self-hosted runners) plus Copilot agents, Pages builds, Dependabot, migrations; at least 74 organizations with failed/stuck workflows |
| Mitigation | Expanded capacity; throttled webhook-triggered work; rolled back; fixed runners acquiring invalid jobs; raised internal rate limits; some manual runner recovery and re-triggered events |
| Follow-ups | More **headroom and autoscaling** for mesh ingress and affected services; **avoid capacity reductions during rollouts**; better saturation monitoring; better load shedding and backlog draining; automatic recovery for self-hosted runners |

**Pattern:** topics 15, 18, 38 — no headroom + rolling update that reduces capacity (`maxUnavailable > 0`). **First check:** saturation (CPU throttling, OOMKilled sidecars) and pod counts during the rollout.

## Case F · 5 October 2026: waiting on timeouts until out of memory

**Summary.** "A partial networking failure on our cloud provider's side disrupted some of the connections between one of our datacenter regions and the cloud-hosted databases that service depends on." "Instead of failing fast, the service waited for those calls to time out. The waiting calls piled up until the service ran out of memory and restarted." Failing over to a healthy region then overloaded that region's instance too, and idle runners shut down and were slow to be replaced.

| Item | Detail |
|---|---|
| Time | Heading: 19:11 UTC, 3 h 38 min (body: ~3 h 45 min) |
| Impact | Actions jobs on GitHub-hosted runners queued much longer than usual; for ~90 minutes most didn't start within five minutes; self-hosted runners unaffected; Pages and some Copilot and billing pages affected |
| Detection | "Alerts flagged errors and slow job assignment, and we declared an incident." |
| Mitigation | Redeploy didn't help; failed over databases and storage to healthy regions one service at a time; added capacity; worked with the cloud provider |
| Follow-ups | Make the job-assignment service handle sudden load; **prioritise runner assignment** when capacity is degraded; change runner behaviour to reduce recovery time; **better detection of partial network failures** between datacenters and cloud services |

**Pattern:** topics 16, 36, 38 — slow dependency without fail-fast timeouts → memory exhaustion; failover moves the overload. **First check:** pending outbound calls/timeouts and memory of the service; health of its dependencies by region.

**Related (6 October 2026):** a surge of migration work let GitHub Enterprise Importer's queries "use all of the database's available connections" — another connection-exhaustion example (topic 30).

---

## In my stack

| GitHub case | My equivalent risk | Prevention |
|---|---|---|
| A cache expiry | Redis key format change in a deploy; mass TTL | Versioned keys, staged rollout, jittered TTL, kill switch |
| B migration | Long `ALTER`/backfill during peak | Off-peak, throttled, circuit-breaker on DB metrics |
| C 32-bit key | Secondary tables/indexes still INT | Inventory **all** columns in the ID path; alerts at 50% |
| D service account | Automated security tooling disabling CI/CD or platform identities | Allowlists; alert on platform-identity auth failures |
| E headroom | `maxUnavailable: 25%` on a near-capacity Deployment | `maxUnavailable: 0`, `maxSurge`, capacity buffer ([K8s 25](../../CKA-Certification-Course-2025/notes/25-deployments.md)) |
| F timeouts | Long default client timeouts + unbounded queues | Short timeouts, bulkheads, circuit breakers, memory limits per queue |

## Follow-up questions

1. **Q: Across these six cases, what's the common thread?**
   A: Most were triggered by a routine change or ordinary event (a deploy, a migration, a peak, an automated review, a partial network failure) hitting a system with little headroom or a hidden limit. The fixes are pattern-level: kill switches, throttling, headroom, limits monitoring, fail-fast timeouts, protection of critical identities.

2. **Q: In case C, why did fixing the primary table not prevent the outage?**
   A: The same ID was stored in a secondary lookup table still using 32 bits. Every place an ID flows needs checking.

3. **Q: In case E, the deploy's code was fine. Why was it the trigger?**
   A: Rolling it out briefly reduced running pods, and the services had no spare capacity. The fix is headroom and rollouts that never reduce capacity, not code changes.

4. **Q: In case F, why did failing over to another region make things worse at first?**
   A: The healthy region inherited the full load (plus backlog) and overloaded too. Failover needs capacity planning and gradual traffic shifts.

5. **Q: In case D, how would you detect this faster?**
   A: Alert specifically on authentication failures of platform service identities, and on any automated action against accounts on a protected list.

6. **Q: In case A, what's the first mitigation?**
   A: Roll back the deploy, as GitHub did; then add a kill switch so the cache behaviour can be turned off without a deploy.

7. **Q: In case B, how would you make migrations safe automatically?**
   A: A migration runner that throttles itself and pauses when DB latency or connection use crosses thresholds (GitHub's stated follow-up).

8. **Q: Why are short public write-ups useful for interviews?**
   A: They show real causes and real follow-ups in a few lines; you can practise explaining trigger → root cause → mitigation → prevention for each.

9. **Q: Which two Part 2 topics appear most often here?**
   A: Capacity/headroom and cascading failure (topics 38, 30/31), followed by limits (33) and automation (12).

10. **Q: If you had to pick one platform-wide improvement from these six, what would it be?**
    A: Saturation monitoring with headroom targets per service, because it would have warned about B, E and F before they failed.

## Source

- GitHub Blog, "GitHub availability report: March 2026": <https://github.blog/news-insights/company-news/github-availability-report-march-2026/>
- GitHub Blog, "GitHub availability report: May 2026": <https://github.blog/news-insights/company-news/github-availability-report-may-2026/>
- GitHub Blog, "GitHub availability report: August 2026": <https://github.blog/news-insights/company-news/github-availability-report-august-2026/>
- GitHub Blog, "GitHub availability report: October 2026": <https://github.blog/news-insights/company-news/github-availability-report-october-2026/>
