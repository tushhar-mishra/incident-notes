# 83 · Drill: "Too Many Connections" from the Database

> **Level:** ★★ intermediate
>
> **You will practise:** a database refusing new connections during a traffic spike, doing the **pool math** across pods (HPA × pool size), finding **who** holds connections, freeing them safely, and fixing the design (**PgBouncer**, HPA limits, timeouts).
>
> **You need:** topics 30, 31, 76.
>
> **Part 4 · On-call drills** · Previous: [82 · Intermittent DNS failures](82-drill-dns-intermittent.md) · Next: [84 · Queue backlog and retries](84-drill-queue-backlog-retries.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **19:05 — PAGE:** `checkout 5xx 12%; logs: "FATAL: sorry, too many clients already"; HPA checkout at max (40 pods)`

**The idea.** A marketing email went out, traffic doubled, HPA scaled checkout from 15 to 40 pods, each with a pool of 20 → up to **800** connections. Postgres `max_connections` is **500**.

```
pods × pool size + other clients  >  max_connections   → new connections refused
scaling out made it WORSE
mitigate: cap HPA / reduce pool size / kill idle-in-transaction / pause non-critical clients
fix: PgBouncer (transaction pooling), pool math in the design, HPA max tied to DB capacity
```

**The one thing to know.** **Do the math out loud in the incident channel.** It immediately shows whether this is a leak, a slow query, or simply too many clients.

> 📖 Read only **Symptoms**, then do the pool math.

---

## Symptoms

- `too many clients already` / `remaining connection slots are reserved for non-replication superuser connections` in app logs.
- HPA at max; DB CPU moderate; connection count flat at the limit.
- Some pods healthy (got connections early), new pods failing readiness.

## Analogy

A restaurant with 500 seats. The manager keeps hiring waiters (pods), and each waiter reserves 20 tables for their customers. After 25 waiters, all tables are reserved — new waiters' customers can't sit, even if half the reserved tables are empty.

## How it happens

Topic 30: each pod's pool opens up to N connections; HPA multiplies pods; PostgreSQL has a hard limit and each connection costs memory. Slow queries or leaks make it worse, but here the main cause is **too many pools**.

### Lab setup (break it)

```bash
docker run -d --name pgc -e POSTGRES_PASSWORD=pw -p 5432:5432 postgres:16 -c max_connections=50 && sleep 5
# Simulate 6 "pods" each opening a pool of 10 idle connections
for p in $(seq 6); do
  for c in $(seq 10); do docker exec -d pgc psql -U postgres -c "SELECT pg_sleep(600);" ; done
done
docker exec pgc psql -U postgres -c "SELECT 1"    # → FATAL: sorry, too many clients already
```

## How to find it

**First — connections vs limit, and who holds them:**

```sql
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
SELECT usename, application_name, client_addr, state, count(*)
FROM pg_stat_activity GROUP BY 1,2,3,4 ORDER BY 5 DESC LIMIT 10;
```
(If you can't connect at all, use a reserved superuser slot — `superuser_reserved_connections` — or the cloud console metrics.)

```
 usename | application_name | client_addr | state  | count
---------+------------------+-------------+--------+-------
 app     | checkout         | 10.0.4.x    | idle   |   410
 app     | checkout         | 10.0.4.x    | active |    60
 app     | reports          | 10.0.9.20   | active |    25
```
- 410 **idle** connections from checkout → pools holding connections they don't use; this is pool sizing × pod count, not a slow query.

**Second — the math:**

```bash
kubectl -n shop get hpa checkout
kubectl -n shop get deploy checkout -o jsonpath='{.spec.template.spec.containers[0].env}' | tr ',' '\n' | grep -i pool
```
`40 pods × pool max 20 = 800` vs `max_connections 500` (minus reserved) → guaranteed failure at high scale.

**Third — rule out leaks/slow queries:** `idle in transaction` counts, oldest transactions (topic 30/31).

## How to fix it

### Immediate

- **Cap HPA** to what the DB can serve: `kubectl -n shop patch hpa checkout -p '{"spec":{"maxReplicas":20}}'` (in Git for GitOps).
- **Reduce pool size** per pod via config (e.g. 20 → 8) and roll out.
- **Terminate idle connections** from the biggest holder (they reconnect as needed):
  ```sql
  SELECT pg_terminate_backend(pid) FROM pg_stat_activity
  WHERE application_name='checkout' AND state='idle' AND state_change < now() - interval '1 minute';
  ```
- Pause non-critical clients (reports).

### Permanent

- **PgBouncer** (transaction pooling) or RDS Proxy between app and DB.
- Pool size from math: `(max_connections − reserved − other clients) ÷ max pods`.
- HPA max documented as tied to DB capacity; alert when HPA is at max.
- `idle_in_transaction_session_timeout`, pool idle timeouts, statement timeouts.

## How to verify the fix

- Connection count well below max (e.g. < 70%) at peak; no "too many clients" errors.
- Pool waiters ≈ 0 in app metrics; latency normal.
- Load test at the next expected peak with max replicas passes.

## Prevention

Pool math in the design review; connection-usage alerts at 80%; pooler by default; load tests that include scale-out.

## Follow-up questions

1. **Q: Root cause?**
   A: Pool size per pod × HPA max replicas exceeded the database's connection limit; scaling out under load made it worse. Contributing: no pooler, no alert on connection usage.
2. **Q: How did you detect it? Why not earlier?**
   A: 5xx page during the spike. An alert at 80% of max_connections, or a design check of pool math, would have caught it.
3. **Q: What did you check first?**
   A: `pg_stat_activity` grouped by client and state, to see whether it's idle pools, slow active queries or leaks.
4. **Q: How did you stop impact?**
   A: Capped HPA, reduced pool size, terminated idle connections, paused reports.
5. **Q: Permanent fix?**
   A: PgBouncer, pool sizing from math, HPA limits linked to DB capacity, timeouts.
6. **Q: How did you confirm?**
   A: Connections below 70% of max at the next peak, no errors, load test passed.
7. **Q: What if reducing pools didn't help (still at the limit)?**
   A: Then connections are held by active slow queries or leaks — check long transactions and blocking (topic 31) and roll back recent changes.
8. **Q: How would you prevent it?**
   A: Make the DB connection budget a first-class capacity limit, with a pooler and alerts.
9. **Q: Why did scaling out make it worse?**
   A: The bottleneck was the database's connection limit; each new pod brought another pool.
10. **Q: What if this happened at 10× traffic?**
    A: You'd need a pooler, read replicas for reads, caching, and load shedding — the primary can't hold 10× connections.
