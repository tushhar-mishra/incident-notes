# 43 · Runaway Cloud Costs

> **Level:** ★★ intermediate
>
> **You will learn:** why a **cost spike is an incident** (it can cost more than an outage), the usual causes (**infinite loops** with serverless or queues, **logging floods**, **NAT gateway and cross-AZ data transfer**, forgotten resources, **autoscaling without limits**, **leaked credentials** used for mining, misconfigured storage requests), how to **find** the runaway with Cost Explorer and usage data in minutes, how to **stop the bleeding**, and **budgets, anomaly detection** and guardrails.
>
> **You need:** topics 01–42. Helpful: [SD 43 · Cost and FinOps](../../System-Design/notes/43-cost-and-finops.md), [GHA 41 · Billing and cost](../../GitHub-Actions-Basics-To-Production/notes/41-billing-limits-and-cost.md).
>
> **Part 2 · Recurring failures · G. Human error, security, cost** · Previous: [42 · Secrets leaked in Git](42-secrets-leaked-in-git.md) · Next: [44 · When monitoring goes dark](44-when-monitoring-goes-dark.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** In the cloud, a bug can spend money **as fast as it can make API calls**. Unlike an outage, nothing looks broken: the system is working very hard at something useless. And the bill arrives later, so you need **alerts**, not invoices, to catch it.

```
Lambda writes to S3 → S3 event triggers the same Lambda → it writes to S3 → ...   (infinite loop)
App logs every request body at DEBUG → 2 TB/day of log ingestion
Pods in AZ-a call a DB in AZ-b through a NAT gateway → per-GB charges on every byte
```

**Picture it.** Leaving a tap running in a house you're not in. Nothing "breaks" — until the water bill comes.

**The one thing to know.** **Set up budgets and cost anomaly alerts on day one**, and treat a cost spike like any other incident: find the change, stop it, then fix it.

**Try it now:** in AWS, open Cost Explorer, group by **Service**, daily, last 30 days. Then group by **Usage type**. Anything you don't recognise?

**You've got it if you can:** find the service and usage type behind a cost spike and stop it within an hour.

> 📖 **That's the core.** Everything below is the **full reference**. Hands-on drill: [87 · Cloud bill 5× overnight](87-drill-cloud-bill-5x.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Alerts | AWS Budgets alert, Cost Anomaly Detection email, credit card/finance message |
| Cost Explorer | One service or usage type jumping (e.g. `NatGateway-Bytes`, `DataTransfer-Regional-Bytes`, `TimedStorage`, `Lambda-GB-Second`, `DataProcessing-Bytes`, CloudWatch `DataProcessing-Bytes` for log ingestion) |
| Metrics | Invocations/requests far above business traffic; queue messages cycling; log volume up 10× |
| Resources | Many instances/GPUs you didn't launch (security!), old snapshots/volumes, idle load balancers, test clusters left running |
| Quota errors | Sometimes the first visible symptom is a quota limit hit by the runaway (topic 35) |

---

## Analogy

The **running tap** (Core).

---

## How it happens

| Cause | Example |
|---|---|
| **Recursive/infinite loops** | Lambda triggered by the S3 bucket it writes to; a queue consumer that re-enqueues failing messages forever; two services calling each other (AWS added automatic detection that stops some Lambda–SQS/SNS/S3 recursive loops) |
| **Logging/metrics floods** | DEBUG logging in prod; high-cardinality metrics (a label with user IDs) → huge ingestion bills |
| **Data transfer** | Traffic through NAT gateways (per-GB processing), cross-AZ chatty services, internet egress, cross-region replication |
| **Autoscaling without limits** | HPA/ASG max set very high; a bug or attack scales to hundreds of nodes |
| **Forgotten resources** | Test environments, unattached EBS volumes, old snapshots, idle RDS, unused Elastic IPs, orphaned LBs |
| **Leaked credentials** | Attackers launch GPU/large instances in every region for crypto-mining (topic 42) |
| **Storage request patterns** | Millions of tiny S3 PUT/GET/LIST requests; S3 lifecycle transitions of millions of small objects |
| **Retry storms** | Retries also multiply cost (requests, invocations) (topic 36) |
| **CI** | Runaway matrix builds; runners never scaled down; artifacts kept forever |

---

## How to find it

**First:** which service, which usage type, starting when?

```bash
aws ce get-cost-and-usage --time-period Start=2026-11-13,End=2026-11-21 \
  --granularity DAILY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE --output json \
  | jq -r '.ResultsByTime[] | .TimePeriod.Start as $d | .Groups[] | select((.Metrics.UnblendedCost.Amount|tonumber) > 50) | "\($d) \(.Keys[0]) \(.Metrics.UnblendedCost.Amount)"'
```
```
2026-11-18 Amazon Elastic Compute Cloud - Compute 410.12
2026-11-19 Amazon Elastic Compute Cloud - Compute 415.80
2026-11-19 EC2 - Other 96.40
2026-11-20 EC2 - Other 2240.55        ← jump
```
- "EC2 - Other" includes NAT gateway, EBS, data transfer. Drill down by `USAGE_TYPE`:

```bash
aws ce get-cost-and-usage --time-period Start=2026-11-19,End=2026-11-21 --granularity DAILY \
  --metrics UnblendedCost --filter '{"Dimensions":{"Key":"SERVICE","Values":["EC2 - Other"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE
```
- e.g. `USE1-NatGateway-Bytes` jumped → huge traffic through the NAT gateway.

**Second:** which resource? Cost allocation **tags** (team/service/env), CUR (Cost and Usage Report) queries in Athena by `line_item_resource_id`, or service metrics:

```bash
# NAT gateway bytes by gateway
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-0abc --statistics Sum --period 3600 \
  --start-time 2026-11-19T00:00:00Z --end-time 2026-11-21T00:00:00Z
```
VPC Flow Logs show which instances/pods send the traffic and to where (often: pulling images or calling S3 without a VPC endpoint).

**Third:** what changed at that time? Deploys, new features, config/log level changes, new jobs, CloudTrail for resource creation (and **unknown** principals → security incident).

---

## How to fix it

### Immediate

- **Stop the runaway:** disable the trigger (remove the S3 event notification, set Lambda reserved concurrency to 0), pause the consumer, scale down, lower log level, terminate unknown instances.
- **Cap:** set ASG/HPA max, Lambda concurrency limits, log retention/ingestion filters.
- If credentials leaked → follow topic 42 first.

### Permanent

- **Budgets** with alerts (actual and forecast) per account/team; **Cost Anomaly Detection** with notifications to a channel.
- **Tags** enforced (SCP/tag policies) so every cost has an owner.
- Architecture: **VPC endpoints** for S3/DynamoDB/ECR (avoid NAT per-GB charges), zone-aware routing, avoid chatty cross-AZ calls where it matters.
- Guardrails: max sizes in IaC modules, policies blocking GPU/large instance types in non-ML accounts, region restrictions via SCPs.
- Logging standards (sampling, levels, cardinality reviews).
- Scheduled shutdown of non-prod; cleanup automation for orphaned resources.

---

## How to verify the fix

- Hourly cost and the specific usage metric back to baseline (Cost Explorer data lags by hours; use service metrics for fast confirmation: invocations, NAT bytes, log ingestion bytes).
- No unknown resources in any region.
- Budget forecast back within plan.

---

## Prevention

- Cost is a **monitored SLI** of the platform: dashboards per team, weekly review.
- Cost checks in PRs for IaC (e.g. Infracost) to see estimated cost changes before apply.
- Game day: "a Lambda recursion in staging" — do alerts fire?

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a new image-processing Lambda wrote thumbnails to the same bucket that triggered it, without a prefix filter → infinite loop. Root cause: event configuration without a filter, no concurrency cap, and no cost anomaly alert in that account.

2. **Q: How did you detect it? Why not earlier?**
   A: Budget alert the next morning. Earlier detection: alarms on invocation rate vs normal, and anomaly detection.

3. **Q: What did you check first?**
   A: Cost Explorer by service and usage type, to find **what** is burning money before guessing.

4. **Q: How did you stop impact?**
   A: Set the Lambda's reserved concurrency to 0 and removed the S3 trigger.

5. **Q: Permanent fix?**
   A: Prefix filters (input/ → output/), concurrency limits, invocation alarms, and anomaly detection on all accounts.

6. **Q: How did you confirm?**
   A: Invocations back to normal immediately; next day's cost back to baseline.

7. **Q: What if you couldn't find the source quickly?**
   A: Use tags and the Cost and Usage Report by resource ID; if still unclear, apply broad caps (lower ASG max, block regions via SCP, reduce concurrency) to stop the spend while investigating — and consider whether it's an attacker.

8. **Q: Why is a cost spike an incident?**
   A: It causes real business damage, sometimes more than an outage, and it often signals something else wrong (a loop, a security breach, a retry storm).

9. **Q: What would you do differently?**
   A: Budgets, anomaly detection and tagging from the first day of every account.

10. **Q: What if the spike was from legitimate traffic growth?**
    A: Then it's capacity planning, not an incident: verify unit cost (cost per request/customer) is stable, optimise the expensive path (caching, VPC endpoints, rightsizing), and update forecasts with finance.
