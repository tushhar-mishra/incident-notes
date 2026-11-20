# 44 · When Monitoring Goes Dark

> **Level:** ★★ intermediate
>
> **You will learn:** why your **observability** often fails **during** the incident you need it for (it shares infrastructure, it gets overloaded by the same event, or it **is** the cause), the difference between "**no alerts**" and "**everything is fine**", **dead man's switches** (watchdog alerts), **out-of-band** and **external** monitoring, how to debug with **no dashboards** (logs on the box, cloud provider metrics, client-side signals), and how to design monitoring that survives.
>
> **You need:** topics 01–43. Helpful: [SD 27 · Observability](../../System-Design/notes/27-observability.md), [SD 50 · Metrics monitoring case](../../System-Design/notes/50-case-metrics-monitoring.md), [K8s 80 · Production observability](../../CKA-Certification-Course-2025/notes/80-production-observability.md).
>
> **Part 2 · Recurring failures · G. Human error, security, cost** · Previous: [43 · Runaway cloud costs](43-runaway-cloud-costs.md) · Next: [45 · AWS S3, February 2017](45-aws-s3-2017.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Monitoring is a system too. It can fail — and it often fails **at the same time** as production, because it runs on the same cluster, network, region or provider. Then the dashboards are empty and **no alerts fire**, which looks exactly like "everything is fine".

```
Production cluster ──┐
                     ├── same network / same region / same DNS / same SSO
Prometheus + Grafana ┘
   → network breaks → production down AND monitoring blind → silence
```

**Picture it.** A smoke detector powered by the same circuit as the kitchen. When an electrical fire trips the circuit, the detector goes silent exactly when you need it.

**The one thing to know.** **Silence must be suspicious.** Use a **dead man's switch** (an alert that is **always firing**; if it stops arriving, your alerting is broken) and at least one **external** check that doesn't share anything with production.

**Try it now:** if your Prometheus/Alertmanager stopped right now, who would notice, and how fast?

**You've got it if you can:** design a watchdog alert and list three ways to debug production with no dashboards.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: [59 · Facebook 2021](59-facebook-bgp-2021.md), [65 · OpenAI 2024](65-openai-kubernetes-2024.md), [70 · Roblox 2021](70-roblox-2021.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Dashboards | Gaps ("No data"), flat lines at zero, stale panels |
| Alerts | Silence during a period when users complain; or a flood of "target down" alerts that hide the real problem |
| Logs | Ingestion delayed by minutes/hours; queries time out |
| Tools | Grafana/Kibana login fails (SSO down); VPN down; monitoring SaaS unreachable |
| Self-inflicted | Monitoring agents themselves overloading systems (OpenAI 2024: a new telemetry service overloaded the Kubernetes API, topic 65); log volume costs exploding (topic 43) |

---

## Analogy

The **smoke detector on the same circuit** (Core).

---

## How it happens

| Cause | Example |
|---|---|
| **Shared fate** | Monitoring runs in the same cluster/region/VPC it monitors; depends on the same DNS, network, service discovery (Roblox 2021: monitoring depended on the failing Consul), or the same SSO |
| **Overload by the incident** | Error storms produce 100× logs; high-cardinality metrics explode; the monitoring pipeline drops data |
| **Monitoring change** | A new agent, scrape config or relabel rule breaks collection silently |
| **Alertmanager/pager failure** | Alert rules fine, but notifications never sent (expired API key to the paging service, misrouted receiver) |
| **Expired credentials/certificates** | Agents can't push to the backend |
| **Blind spots** | Only server-side metrics; the failure is in DNS/CDN/LB in front of the servers |

### Dead man's switch

```yaml
# Prometheus rule: always firing
- alert: Watchdog
  expr: vector(1)
  labels: { severity: none }
  annotations: { summary: "Alerting pipeline heartbeat" }
```
Alertmanager sends it to an **external** heartbeat service (e.g. a paging tool's heartbeat feature or a dedicated dead-man's-switch service). If the heartbeat **stops**, the external service pages you: "your monitoring is broken". (The kube-prometheus stack ships a `Watchdog` alert for exactly this purpose.)

---

## How to find it

**First:** is it **production** that's fine, or **monitoring** that's blind? Check from outside.

```bash
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' https://shop.example.com/healthz
```
- Plus: external synthetic checks, customer support tickets, social media, business metrics (orders per minute from the payment provider's dashboard).

**Second:** is the monitoring pipeline working?

```bash
kubectl -n monitoring get pods
kubectl -n monitoring logs statefulset/prometheus-k8s -c prometheus --tail=50 | grep -iE "error|scrape|out of"
# Prometheus self-metrics
#   up == 0 for many targets?  prometheus_tsdb_head_samples_appended_total flat?
#   alertmanager_notifications_failed_total increasing?
```

**Third:** debug production **without** the dashboards:

| Source | Example |
|---|---|
| **On the host** | `journalctl`, `/var/log`, `top`, `ss`, `dmesg` via SSH/SSM/serial console |
| **Kubernetes directly** | `kubectl get pods/events/logs` (if the API works) |
| **Cloud provider metrics** | CloudWatch/LB metrics, RDS metrics — different infrastructure from your Prometheus |
| **Client side** | CDN analytics, browser RUM (Real User Monitoring), mobile crash reporting, external synthetics |
| **Business signals** | Payment provider dashboard, order counts from the DB, support ticket rate |

---

## How to fix it

### Immediate

- **Assume impact until proven otherwise**; use external signals to estimate it.
- Restore the monitoring pipeline if it's quick (restart Prometheus, scale the log pipeline, drop the noisy log source).
- If monitoring is **causing** load (agent bug), turn it off.
- Use the cloud provider's metrics and direct access to make decisions.

### Permanent

- **External, independent monitoring**: synthetics from outside your infrastructure, on a different provider.
- **Meta-monitoring**: a small separate Prometheus (or the SaaS) that monitors your main monitoring; Watchdog heartbeat to an external service.
- **Separate failure domains**: monitoring cluster outside production clusters/regions; local break-glass accounts that don't need SSO.
- **Back-pressure** in log/metric pipelines (buffering, sampling, dropping debug logs first).
- Treat monitoring changes as **production changes** (review, staged rollout).
- Test alerts end to end regularly (fire a test page).

---

## How to verify the fix

- Dashboards show continuous data; no gaps.
- Watchdog heartbeat arriving; a test alert reaches the on-call phone.
- External synthetics agree with internal metrics.

---

## Prevention

- Quarterly "blind" drill (topic 89): solve an incident with Grafana turned off.
- Inventory of what monitoring depends on (DNS, SSO, network, region, storage) and of independent fallbacks.
- Alert on monitoring health: scrape failures, ingestion lag, notification failures.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a network change broke traffic within one region; Prometheus and Alertmanager ran in the same cluster and couldn't reach PagerDuty either, so no page was sent for 40 minutes. Root cause: monitoring shared fate with production, and no external heartbeat existed.

2. **Q: How did you detect it? Why not earlier?**
   A: Customers reported it. An external synthetic check or a missing Watchdog heartbeat would have paged within minutes.

3. **Q: What did you check first?**
   A: Whether the service was really down from outside, because silence from monitoring proves nothing.

4. **Q: How did you stop impact?**
   A: Used CloudWatch LB metrics and direct `kubectl` access to find and revert the network change.

5. **Q: Permanent fix?**
   A: Watchdog to an external heartbeat service, external synthetics, and a monitoring stack in a separate failure domain.

6. **Q: How did you confirm?**
   A: Simulated a monitoring outage: the heartbeat service paged within 5 minutes.

7. **Q: What if you also had no access to the hosts (SSO down)?**
   A: Use break-glass accounts that don't depend on SSO, cloud serial consoles/SSM, and the provider's console. These must be prepared and tested in advance.

8. **Q: Can monitoring cause outages?**
   A: Yes: agents overloading APIs (OpenAI 2024), sidecars consuming memory, log shipping saturating disks or networks. Monitoring changes need the same staged rollout as anything else.

9. **Q: What would you do differently?**
   A: Ask "what does our monitoring depend on?" and remove shared dependencies for the alerting path.

10. **Q: What if this happened at 10× traffic?**
    A: The incident would produce even more logs and metrics, so the pipeline is likely to drop data. Sampling and prioritised pipelines (errors and SLO metrics first) keep the important signals flowing.
