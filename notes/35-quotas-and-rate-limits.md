# 35 · Quotas and Rate Limits

> **Level:** ★★ intermediate
>
> **You will learn:** the limits you hit in production that **aren't** CPU or memory: **cloud service quotas** (instances, IPs, load balancers, Lambda concurrency), **cloud API rate limits** (throttling of control-plane calls), **third-party API limits** (payment, email, GitHub, Docker Hub), and **your own** rate limits; how they show up (**429**, `ThrottlingException`, `LimitExceeded`), why they hit hardest during **scale-up and recovery**, and how to monitor and raise them **before** you need them.
>
> **You need:** topics 01–34. Helpful: [SD 25 · Rate limiting and backpressure](../../System-Design/notes/25-rate-limiting-and-backpressure.md), [GHA 41 · Billing, limits and cost](../../GitHub-Actions-Basics-To-Production/notes/41-billing-limits-and-cost.md).
>
> **Part 2 · Recurring failures · F. Overload and cascades** · Previous: [34 · Backups that don't restore](34-backups-that-dont-restore.md) · Next: [36 · Retry storms](36-retry-storms.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every cloud account and every external API has **limits**: how many of a thing you can have (**quota**) and how fast you can ask (**rate limit**). You don't notice them on normal days. You hit them on the worst days: traffic spikes, big scale-ups, disaster recovery, when everybody retries at once.

```
Normal day:   launch 5 instances/hour     quota: 200 vCPUs     ✅
DR day:       launch 300 instances now    → "VcpuLimitExceeded" ❌
Spike:        call payment API 500/s      limit: 100/s         → 429 Too Many Requests ❌
```

**Picture it.** A credit card. You use it every day without thinking about the limit. On the day of an emergency (car breaks down abroad), you discover the limit — at the worst possible moment.

**The one thing to know.** **Check quotas for your disaster-recovery and peak scenarios in advance.** Quota increases can take hours or days to approve.

**Try it now:** `aws service-quotas list-service-quotas --service-code ec2 --query 'Quotas[?contains(QuotaName,`On-Demand`)].[QuotaName,Value]' --output table`

**You've got it if you can:** list the quotas that would block your service from doubling in size, and explain how to handle 429s safely.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| AWS | `VcpuLimitExceeded`, `InstanceLimitExceeded`, `LimitExceededException`, `ThrottlingException: Rate exceeded`, `RequestLimitExceeded`, `TooManyRequestsException` (Lambda concurrency) |
| Kubernetes on cloud | Cluster autoscaler can't add nodes; load balancer controller can't create LBs; pods can't get IPs (subnet IP exhaustion) |
| Third-party APIs | HTTP **429 Too Many Requests** with a `Retry-After` header; `X-RateLimit-Remaining: 0` |
| CI/CD | `toomanyrequests` from Docker Hub; GitHub API secondary rate limits; runner concurrency limits |
| Terraform | `Error: ... ThrottlingException` during big applies; quota errors creating resources |
| Users | Features that depend on the limited API fail; scale-up stalls during a spike |

---

## Analogy

The **credit card limit** (Core). Rate limits are like a cash machine that only gives out a few withdrawals per minute.

---

## How it happens

| Type | Example | Typical trigger |
|---|---|---|
| **Resource quotas** | EC2 vCPUs per family, Elastic IPs, VPCs, NAT gateways, LBs, EBS volume storage | Scale-up, DR into a new region (quotas are **per region**), new environment |
| **Control-plane API rate limits** | EC2 `Describe*` calls, Route 53 change calls, IAM | Autoscalers, Terraform, controllers polling too often; many tools sharing one account's limit |
| **Concurrency limits** | Lambda account concurrency (regional default 1,000; can be raised) | Traffic spike; one function starving others |
| **Network limits** | Subnet free IPs; NAT gateway ports per destination; connection tracking | Pods using one VPC IP each; many connections to one destination |
| **Third-party** | Payment/email/SMS provider rate limits; Docker Hub pull limits; GitHub API limits | Spikes, retry storms, batch jobs |
| **Your own** | Your API gateway limits per client | A legitimate big customer gets throttled |

**Why worst-day timing:** during recovery, **everyone** does the same thing at once: thousands of instances launch, clients reconnect, controllers relist, retries fire. Limits that were never close get hit.

---

## How to find it

**First:** read the exact error — it names the limit.

```bash
kubectl -n kube-system logs deploy/cluster-autoscaler | grep -i -E "limit|quota|exceed" | tail
aws ec2 describe-instances ... 2>&1 | tail -1
```
```
An error occurred (VcpuLimitExceeded) when calling the RunInstances operation:
You have requested more vCPU capacity than your current vCPU limit of 256 allows
for the instance bucket that the specified instance type belongs to.
```

**Second:** current usage vs quota.

```bash
aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A   # Running On-Demand Standard instances (vCPUs)
aws cloudwatch get-metric-statistics --namespace AWS/Usage --metric-name ResourceCount \
  --dimensions Name=Service,Value=EC2 Name=Type,Value=Resource Name=Resource,Value=vCPU Name=Class,Value=Standard/OnDemand \
  --start-time "$(date -u -d '-1 hour' +%FT%TZ)" --end-time "$(date -u +%FT%TZ)" --period 300 --statistics Maximum
# Subnet IPs
aws ec2 describe-subnets --query 'Subnets[].[SubnetId,AvailableIpAddressCount,CidrBlock]' --output table
```

API throttling: CloudTrail events with `errorCode` = `ThrottlingException` / `Client.RequestLimitExceeded`, grouped by `userAgent` → **who** is using up the shared rate limit.

**Third:** third-party limits: response headers.

```bash
curl -sI -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/rate_limit | grep -i x-ratelimit
```
```
x-ratelimit-limit: 5000
x-ratelimit-remaining: 0
x-ratelimit-reset: 1763640000
```

---

## How to fix it

### Immediate

- **Request a quota increase** (Service Quotas console/API; for urgent cases, open a high-severity support case).
- **Use what you have:** other instance families/sizes (separate vCPU buckets), other regions/zones, Spot vs On-Demand.
- **Reduce demand:** stop non-critical workloads, slow down batch jobs, pause noisy controllers/automation that burn API rate limits.
- **429s from a provider:** respect `Retry-After`, back off with jitter (topic 36), queue requests, and degrade the feature.
- Subnet IP exhaustion: add subnets/secondary CIDR, enable prefix delegation (EKS VPC CNI).

### Permanent

- **Quota review** for peak and DR scenarios (e.g. "can we run 2× in another region?") — pre-approve increases.
- Monitor quota usage (Service Quotas + CloudWatch usage metrics, AWS Trusted Advisor; equivalents in GCP/Azure) with alerts at 70–80%.
- Spread automation across accounts or reduce polling; use caching for Describe calls.
- Clients: backoff + jitter, honour `Retry-After`, client-side rate limiting, retry budgets.
- Mirror images and dependencies (topic 20) so registry limits don't block recovery.

---

## How to verify the fix

- New quota value visible; usage comfortably below it.
- Scale-up completes; no throttling errors in CloudTrail/logs.
- 429 rate back to near zero; queued work drained.

---

## Prevention

- Quota dashboards per account/region; alerts before limits.
- DR runbooks include quota checks for the target region.
- Load tests include the scale-up path (not only steady state).
- Contractual limits with critical vendors documented, with a contact for emergency increases.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: during a regional failover, we needed 3× our usual capacity in the DR region, but its vCPU quota was set for normal DR tests. Root cause: DR plan never validated quotas at full scale.

2. **Q: How did you detect it? Why not earlier?**
   A: Cluster autoscaler errors and Pending pods during failover. DR drills used small scale, so the limit never appeared.

3. **Q: What did you check first?**
   A: The exact error from the autoscaler/cloud API, because quota errors name the limit and the bucket.

4. **Q: How did you stop impact?**
   A: Used other instance families with free quota, requested an urgent increase, and shed non-critical workloads.

5. **Q: Permanent fix?**
   A: Pre-approved quotas sized for full DR, quota usage alerts, and DR drills at realistic scale.

6. **Q: How did you confirm?**
   A: Quota values updated; a scaled DR test reached target capacity.

7. **Q: What if the quota increase took days?**
   A: Use alternatives: different instance types/families, Spot, another region, fewer replicas of non-critical services, and degrade features to fit inside the capacity you have.

8. **Q: How do you handle a 429 from a third-party API correctly?**
   A: Respect `Retry-After`, retry with exponential backoff and jitter, limit retries, queue non-urgent work, and degrade the feature; never retry immediately in a tight loop.

9. **Q: What would you do differently?**
   A: Treat quotas as capacity: track them like CPU and disk, including the DR region.

10. **Q: What if many internal tools share one AWS account's API rate limit?**
    A: One noisy tool (a misconfigured controller polling Describe APIs) throttles everyone, including the autoscaler during an incident. Find it via CloudTrail user agents, reduce its polling, and separate critical automation into its own account where possible.
