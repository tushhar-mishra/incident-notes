# 79 · Drill: p99 Latency Up, Nothing Deployed

> **Level:** ★★ intermediate
>
> **You will practise:** the harder "**nothing changed**" incident: widening the search beyond deploys (config, traffic, dependencies, data, platform), using **RED/USE** and **follow the request** to find the slow hop, and mitigating before you're sure.
>
> **You need:** topics 08, 15, 30, 31, 38, 76.
>
> **Part 4 · On-call drills** · Previous: [78 · Disk at 95% on the DB host](78-drill-disk-95-database.md) · Next: [80 · CrashLoopBackOff after a ConfigMap change](80-drill-crashloop-after-configmap.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **02:40 — PAGE:** `api-latency-slo-burn — p99 /api/orders 2.8s (SLO 500ms), errors 0.3%, started ~02:10`

**The idea.** No deploy in 24 hours. Latency **ramped** over 30 minutes. Find **which hop** got slow and **why now**.

```
Shape: ramp (not step) → growth, saturation, backlog, data — not usually a deploy
Follow the request: LB → api pods → cache → DB → external API — which hop's latency moved first?
Check saturation (USE): CPU throttling, pool waiters, DB locks/slow queries, queue depth
Candidates "that aren't deploys": traffic mix change, a batch job, stats/plan flip, dependency slowdown, cache hit drop
```

**The one thing to know.** "Nothing was deployed" means **look harder at other changes**: cron jobs, data growth, a big customer, a dependency, autovacuum/ANALYZE, cache evictions.

> 📖 Read only **Symptoms**, then write three hypotheses and one test for each.

---

## Symptoms

- p99 latency on `/api/orders` rising since ~02:10; p50 only slightly up; errors low but rising.
- No deploy, flag or Terraform change in the last 24 hours.
- Request rate normal for the time of night.

## Analogy

Your commute is 30 minutes slower and there are no roadworks announced. You check: more cars? a closed lane? an accident on one junction? You look at each part of the route to find the slow stretch.

## How it happens

Typical non-deploy causes:
- A **scheduled job** (report, backfill, backup) competing for the database.
- A **query plan flip** after ANALYZE (topic 31, Clerk 2026 topic 72).
- **Connection pool waits** because DB queries got slower (topic 30).
- **Cache hit rate drop** (evictions, a key change by another service) (topic 37).
- **Dependency** slowness (payment provider, another team's service).
- **Noisy neighbour / CPU throttling** on nodes (topic 15).
- A **large customer** doing something unusual (bulk export).

### Lab setup (break it)

```bash
docker run -d --name pgl -e POSTGRES_PASSWORD=pw -p 5432:5432 postgres:16 && sleep 5
docker exec pgl psql -U postgres -c "CREATE TABLE orders AS SELECT g id, g%1000 cust, now() ts FROM generate_series(1,2000000) g; CREATE INDEX ON orders(cust);"
# "API": a fast indexed query in a loop, measuring latency
( while true; do s=$(date +%s%N); docker exec pgl psql -U postgres -qtc "SELECT count(*) FROM orders WHERE cust=42;" >/dev/null; echo "$(( ($(date +%s%N)-s)/1000000 )) ms"; sleep 0.5; done ) &
# The hidden "batch job" that starts at night: heavy sequential scans
docker exec -d pgl sh -c 'while true; do psql -U postgres -qc "SELECT sum(id) FROM orders o1 JOIN orders o2 USING (cust) WHERE o1.id < 20000;" ; done'
```

## How to find it

**First — confirm scope and shape:** which endpoints, all pods or some, p50 vs p99, ramp vs step. Compare with the same time yesterday.

**Second — follow the request (which hop moved first?):**

```
# Prometheus examples
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{route="/api/orders"}[5m])))
histogram_quantile(0.99, sum by (le, peer) (rate(client_request_duration_seconds_bucket{app="api"}[5m])))   # per dependency
hikaricp_connections_pending{app="api"}                                    # pool waiters
rate(container_cpu_cfs_throttled_periods_total{pod=~"api-.*"}[5m])        # throttling
```
Traces: open a slow `/api/orders` trace — which span grew?

**Third — the database (the usual suspect):**

```sql
SELECT pid, now()-query_start AS age, state, wait_event_type, left(query,60)
FROM pg_stat_activity WHERE state <> 'idle' ORDER BY age DESC LIMIT 5;
```
```
  pid  |   age    | state  | wait_event_type | left
-------+----------+--------+-----------------+------------------------------------------
 81234 | 00:31:02 | active | IO              | SELECT sum(id) FROM orders o1 JOIN orders
```
- A 31-minute heavy query started around 02:10 = the hidden batch job.

```sql
SELECT left(query,50), calls, round(mean_exec_time) ms FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;
```
And check: `pg_stat_user_tables.last_autoanalyze` (plan flip?), cron/Kubernetes CronJobs that started at 02:00.

```bash
kubectl get cronjobs -A; kubectl get jobs -A --sort-by=.status.startTime | tail
```

## How to fix it

### Immediate

- **Stop or pause the competing job** (cancel the query: `SELECT pg_cancel_backend(81234);`, suspend the CronJob: `kubectl patch cronjob reports -p '{"spec":{"suspend":true}}'`).
- If a plan flip: `ANALYZE orders;` (topic 31).
- If a dependency: timeouts/circuit breaker/feature off (topic 38).
- If saturation in the app: scale out (only if the bottleneck is the app) or remove CPU limits.

### Permanent

- Move heavy reporting to a **read replica** or warehouse; schedule off-peak with throttling.
- Per-query latency alerts; per-dependency latency dashboards.
- Statement timeouts for batch roles.

## How to verify the fix

- p99 back under SLO within minutes of stopping the job; pool waiters 0; DB CPU/IO normal.
- The job runs safely next night (on a replica / throttled) without affecting p99.

## Prevention

Change calendars that include **jobs**, not just deploys; resource isolation for batch work; dashboards that break latency down by dependency.

## Follow-up questions

1. **Q: Root cause?**
   A: A nightly reporting job ran heavy queries on the primary database, saturating I/O and slowing the hot query path. Contributing: no workload isolation, no per-query alerting.
2. **Q: How did you detect it? Why not earlier?**
   A: Latency burn alert after 30 minutes of ramp. A DB saturation or long-running-query alert would have fired earlier.
3. **Q: What did you check first?**
   A: The graph shape and per-hop latency, to find which component slowed first — the DB.
4. **Q: How did you stop impact?**
   A: Cancelled the job's query and suspended the CronJob.
5. **Q: Permanent fix?**
   A: Reports on a read replica, statement timeouts, off-peak throttled scheduling, alerts.
6. **Q: How did you confirm?**
   A: p99 back under SLO; DB metrics normal; next night's run didn't affect latency.
7. **Q: What if stopping the job didn't help?**
   A: Next hypotheses: plan flip (re-ANALYZE, compare EXPLAIN), dependency slowness (per-peer latency), cache hit drop, node throttling. Keep following the request.
8. **Q: How would you prevent it?**
   A: Isolate batch from OLTP, include scheduled jobs in "what changed?", alert on long queries.
9. **Q: What would you do differently?**
   A: Check scheduled jobs and DB activity in the first five minutes when the shape is a ramp and nothing was deployed.
10. **Q: What if you had no access to the database?**
    A: Use what you have: app-side pool metrics and per-query timings in traces, the cloud provider's DB metrics (CPU, IOPS, connections), and ask the DB owner on call — while mitigating on the app side (shed load, disable the heaviest endpoint).
