# 84 · Drill: Queue Backlog Growing, Retries Climbing

> **Level:** ★★★ advanced
>
> **You will practise:** an async pipeline falling behind: a message queue backlog grows, consumers keep **retrying failed messages**, a **poison message** blocks progress, the downstream dependency is overloaded by retries, and you must **stop the storm**, then **drain the backlog** without causing a second overload.
>
> **You need:** topics 36, 37, 38, 76. Helpful: [SD 21 · Queues, pub/sub, streaming](../../System-Design/notes/21-queues-pubsub-streaming.md), [SD 22 · Async patterns](../../System-Design/notes/22-async-patterns.md).
>
> **Part 4 · On-call drills** · Previous: [83 · "Too many connections"](83-drill-too-many-connections.md) · Next: [85 · Terraform plan destroys the prod DB](85-drill-terraform-destroy-prod-db.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **01:30 — PAGE:** `orders-queue age of oldest message 42 min (SLO 5 min); consumer error rate 65%; email-provider 429s`

**The idea.** Consumers fail on messages, put them back, and retry immediately. Retries flood the email provider (429s), so even good messages fail. The queue grows; the oldest message gets older.

```
check: what's failing (one poison message? the dependency? our consumer?)
stop the storm: pause/slow consumers, backoff, DLQ the poison messages, respect 429 Retry-After
drain: scale consumers gradually within the dependency's rate limit, newest-critical first
```

**The one thing to know.** **Scaling consumers during a retry storm makes it worse.** First reduce load on the struggling dependency, then drain at a rate it can handle.

> 📖 Read only **Symptoms**, then write your first three checks.

---

## Symptoms

- Queue depth and "age of oldest message" climbing (SQS `ApproximateAgeOfOldestMessage`, Kafka consumer lag, RabbitMQ ready messages).
- Consumer error rate high; the same message IDs processed many times.
- Downstream provider returns **429 Too Many Requests**.
- Users: order confirmation emails delayed by an hour.

## Analogy

A post office sorting line. One parcel is jammed (poison message). The sorter keeps grabbing it, failing, and putting it back at the front. Meanwhile, the courier (email provider) is told "stop sending us so much!" and the queue of parcels grows out the door.

## How it happens

- A **poison message** (malformed payload) always fails; with immediate redelivery and no max-receive limit, it's retried forever and can block ordered processing.
- **Dependency degradation** (provider slow / rate limiting) → failures → retries **without backoff** → more requests → more 429s (topic 36).
- **Autoscaling on queue depth** adds consumers → even more requests to the struggling dependency.

### Lab setup (break it)

Use a local RabbitMQ or Redis list, or simulate:

```bash
docker run -d --name redisq -p 6379:6379 redis:7
# Producer: 50 msgs/s, one poison message
docker exec -d redisq sh -c 'i=0; while true; do i=$((i+1)); redis-cli RPUSH q "{\"id\":$i,\"email\":\"u$i@x\"}" >/dev/null; [ $i -eq 100 ] && redis-cli RPUSH q "{bad json" >/dev/null; sleep 0.02; done'
# Consumer: pops, "fails" on poison and when the "provider" is rate-limited, pushes back immediately (no backoff)
docker exec -d redisq sh -c 'while true; do m=$(redis-cli LPOP q); [ -z "$m" ] && continue; case "$m" in *bad*) redis-cli LPUSH q "$m" >/dev/null;; *) sleep 0.05 ;; esac; done'   # slower than the producer → backlog grows
watch -n 2 'docker exec redisq redis-cli LLEN q'
```

## How to find it

**First — size and trend:** depth and oldest-message age; incoming vs processed rate.

```bash
aws sqs get-queue-attributes --queue-url $Q --attribute-names ApproximateNumberOfMessages ApproximateAgeOfOldestMessage
kafka-consumer-groups.sh --bootstrap-server $B --describe --group orders-consumers   # LAG per partition
```

**Second — what's failing and why?** Group consumer errors by type and message ID.

```bash
kubectl -n shop logs deploy/order-consumer --since=10m | grep -E "ERROR|retry" | awk '{print $NF}' | sort | uniq -c | sort -rn | head
```
```
  4321 status=429 provider=email
    912 msg_id=100 error=JSONDecodeError      ← same message failing over and over (poison)
```

**Third — are we hammering the dependency?** Requests/s to the provider vs its limit; `Retry-After` headers; consumer replica count (HPA on queue depth?).

```bash
kubectl -n shop get hpa order-consumer
```

## How to fix it

### Immediate

1. **Reduce pressure on the provider:** scale consumers **down** (or pause), disable HPA scale-up temporarily, honour `Retry-After`.
2. **Remove poison messages:** move them to a **dead-letter queue (DLQ)** (SQS redrive policy with `maxReceiveCount`, RabbitMQ DLX, Kafka error topic).
3. **Turn on backoff** (config flag) for retries: exponential + jitter (topic 36).
4. **Drain gradually:** increase consumers step by step while provider errors stay low; prioritise time-sensitive messages if you have separate queues.

### Permanent

- DLQ with `maxReceiveCount` (e.g. 5) on every queue; alerts on DLQ depth.
- Retries with exponential backoff, jitter and a cap; **idempotent** consumers (safe to reprocess).
- Client-side rate limiting to the provider's limit; a circuit breaker.
- Autoscaling on queue age **with a max** that respects downstream limits.
- Validate messages at the producer (schema) so poison messages are rejected early.

## How to verify the fix

- Oldest-message age falling steadily; depth trending to normal; error rate low; provider 429s ≈ 0.
- DLQ contains only the bad messages (inspect and replay after fixing if they're valid).
- Users receive delayed emails; no duplicates (idempotency).

## Prevention

Queue SLOs (age of oldest message), DLQs everywhere, retry policies reviewed, downstream limits documented in the consumer config.

## Follow-up questions

1. **Q: Root cause?**
   A: A poison message retried forever without a DLQ, plus retries without backoff that overloaded a rate-limited provider; autoscaling on queue depth added more pressure.
2. **Q: How did you detect it? Why not earlier?**
   A: Oldest-message-age SLO alert. An alert on consumer error rate or DLQ depth would fire earlier.
3. **Q: What did you check first?**
   A: What exactly was failing (grouping errors), because a poison message, a dependency problem and a consumer bug need different fixes.
4. **Q: How did you stop impact?**
   A: Scaled consumers down, DLQ'd the poison message, enabled backoff, then drained gradually.
5. **Q: Permanent fix?**
   A: DLQs, backoff + jitter, idempotent consumers, client-side rate limiting, bounded autoscaling, producer validation.
6. **Q: How did you confirm?**
   A: Backlog age decreasing to normal; provider error rate low; no duplicate side effects.
7. **Q: What if the backlog keeps growing even after the storm stops?**
   A: Incoming rate exceeds what the provider allows; you need more provider capacity (raise limits with the vendor), a second provider, or to shed/aggregate low-priority messages.
8. **Q: How would you prevent it?**
   A: Treat queues as systems with SLOs, limits and dead-letter paths from day one.
9. **Q: Why is idempotency important here?**
   A: At-least-once delivery plus retries means messages can be processed twice; idempotent consumers make that safe (no duplicate emails or charges).
10. **Q: What if this happened during a big sale (10× messages)?**
    A: The provider limit becomes the hard ceiling; queue messages for later, prioritise critical ones (receipts over marketing), and pre-arrange higher limits.
