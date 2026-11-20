# 87 · Drill: Cloud Bill 5× Overnight

> **Level:** ★★ intermediate
>
> **You will practise:** a **cost incident**: finding the service and usage type behind a spike with Cost Explorer, confirming with **usage metrics** (which update faster than the bill), stopping the runaway (recursive trigger, logging flood, NAT traffic), and adding budgets and guardrails.
>
> **You need:** topics 43, 76. Helpful: [SD 43 · Cost and FinOps](../../System-Design/notes/43-cost-and-finops.md).
>
> **Part 4 · On-call drills** · Previous: [86 · AWS key pushed to a public repo](86-drill-aws-key-leaked.md) · Next: [88 · One AZ is gone](88-drill-az-down.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **08:30 — Email:** `AWS Cost Anomaly Detection: $3,912 above expected spend for "AWS Lambda" + "Amazon S3" in account prod-media (last 24h)`

**The idea.** Cost data lags by hours, so **confirm with service metrics** and act fast: every hour costs money.

```
Cost Explorer: which service, which usage type, which day/hour?
Service metrics: invocations, requests, bytes — is it still happening right now?
What changed: deploy, trigger, config, log level, new job — or an attacker?
Stop: disable trigger / concurrency 0 / scale down / lower log level
Guard: budgets, anomaly alerts, concurrency limits, prefix filters, tags
```

**The one thing to know.** **Stop the spend first, then optimise.** A runaway loop can cost more per hour than a week of normal usage.

> 📖 Read only **Symptoms**, then write your first three checks.

---

## Symptoms

- Anomaly/budget alert; Cost Explorer shows Lambda and S3 jumping from ~$40/day to ~$4,000/day.
- Lambda invocations metric: 2,000/s (normal 5/s); S3 PUT requests spiking.
- No customer-visible errors (the system is "working hard").

## Analogy

A tap left running in an empty house, or a toilet that keeps refilling because the float is broken: nothing looks wrong until the water bill arrives.

## How it happens

Topic 43. Here: a new thumbnail Lambda triggered by `s3:ObjectCreated` on bucket `media` writes thumbnails **back into the same bucket** without a prefix filter → each write triggers the function again → infinite loop. (AWS can detect and stop some recursive loops involving Lambda with SQS, SNS and S3, but don't rely on it: design so loops can't happen.)

### Lab setup (safe)

Do it as a **tabletop**, or in a sandbox account with a strict **budget alert**, a Lambda **reserved concurrency of 1**, and a short time limit. Simpler local version: a script that watches a directory and writes a file into the same directory each time it sees one — watch the file count explode, then stop it.

## How to find it

**First — which service and usage type:**

```bash
aws ce get-cost-and-usage --time-period Start=2026-11-18,End=2026-11-21 --granularity DAILY \
  --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[].Groups[?Metrics.UnblendedCost.Amount>`100`].[Keys[0],Metrics.UnblendedCost.Amount]'
aws ce get-cost-and-usage --time-period Start=2026-11-19,End=2026-11-21 --granularity DAILY \
  --metrics UnblendedCost --filter '{"Dimensions":{"Key":"SERVICE","Values":["AWS Lambda"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE
```

**Second — is it happening now? (service metrics are near real-time)**

```bash
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Invocations \
  --dimensions Name=FunctionName,Value=make-thumbnail --statistics Sum --period 300 \
  --start-time "$(date -u -d '-2 hours' +%FT%TZ)" --end-time "$(date -u +%FT%TZ)"
```
- ~600,000 invocations per 5 minutes and climbing → active loop.

**Third — why?**

```bash
aws s3api get-bucket-notification-configuration --bucket media
aws lambda get-function-configuration --function-name make-thumbnail --query 'Environment.Variables'
```
```
{"LambdaFunctionConfigurations":[{"LambdaFunctionArn":"...:make-thumbnail","Events":["s3:ObjectCreated:*"]}]}   ← no prefix filter
{"OUTPUT_BUCKET":"media","OUTPUT_PREFIX":"thumbnails/"}                                                  ← writes back to same bucket
```
Also check CloudTrail for who created it and when (and rule out a leaked key, topic 86).

## How to fix it

### Immediate

```bash
aws lambda put-function-concurrency --function-name make-thumbnail --reserved-concurrent-executions 0   # stop now
aws s3api put-bucket-notification-configuration --bucket media --notification-configuration '{}'        # remove trigger
```
Then clean up generated junk objects (lifecycle rule or batch delete), and re-enable the feature properly.

### Permanent

- Trigger only on `uploads/` prefix; write to `thumbnails/` (or a different bucket).
- Reserved concurrency caps on functions; alarms on invocation rate vs baseline.
- **AWS Budgets** (actual + forecast) and **Cost Anomaly Detection** for every account, routed to the on-call channel.
- Cost estimation in PRs (Infracost); mandatory cost-allocation tags.

## How to verify the fix

- Invocations back to baseline within minutes; S3 request rate normal.
- Next day's cost back to baseline in Cost Explorer.
- A test upload produces exactly one thumbnail.

## Prevention

Budgets and anomaly alerts everywhere, concurrency limits, event filters, cost review in design and PRs.

## Follow-up questions

1. **Q: Root cause?**
   A: An S3 event trigger without a prefix filter on a bucket the function also wrote to, creating a recursive loop; no concurrency cap and no fast cost/invocation alert.
2. **Q: How did you detect it? Why not earlier?**
   A: Cost anomaly email the next morning. An invocation-rate alarm would have fired within minutes.
3. **Q: What did you check first?**
   A: Which service/usage type, then live metrics to confirm it was still running.
4. **Q: How did you stop impact?**
   A: Reserved concurrency 0 and removed the trigger.
5. **Q: Permanent fix?**
   A: Prefix filters/separate buckets, concurrency limits, invocation alarms, budgets and anomaly detection.
6. **Q: How did you confirm?**
   A: Invocations normal immediately; next-day cost normal; test upload produced one thumbnail.
7. **Q: What if you couldn't find the source quickly?**
   A: Apply broad brakes while investigating: lower concurrency on recently changed functions, SCPs to block new resource launches, scale down; check for a security compromise.
8. **Q: How would you prevent it?**
   A: Guardrails that cap spend velocity (concurrency, quotas) plus fast alerts on usage, not just on the bill.
9. **Q: What would you do differently?**
   A: Route anomaly alerts to the paging channel for production accounts, not just email.
10. **Q: What if the spike were NAT gateway data processing instead?**
    A: Find the source with VPC Flow Logs / NAT metrics; usually S3/ECR/DynamoDB traffic that should go through **VPC endpoints**, or a misbehaving client downloading repeatedly.
