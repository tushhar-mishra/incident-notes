# 89 · Drill: No Logs, No Dashboards — Monitoring Is Down Too

> **Level:** ★★★ advanced
>
> **You will practise:** debugging **blind**: users report problems, Grafana shows "No data", alerts are silent. Establishing impact from **outside**, using **direct access** (kubectl, SSH/SSM, cloud provider metrics) instead of dashboards, and restoring observability without making things worse.
>
> **You need:** topics 08, 44, 76.
>
> **Part 4 · On-call drills** · Previous: [88 · One AZ is gone](88-drill-az-down.md) · Next: [90 · Capstone game day](90-drill-game-day-capstone.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📞 **09:40 — Phone call from support:** "Customers say the app is very slow and some can't log in. Your status page is green." You open Grafana: **No data** for the last 25 minutes. No alerts fired.

**The idea.** Silence is not health. First establish **real impact from outside**, then debug with tools that **don't depend on the broken monitoring**, and in parallel restore the monitoring.

```
impact from outside: curl from your laptop / external synthetics / support tickets / business metrics
direct tools: kubectl get/describe/logs/events/top, node SSH/SSM, cloud provider metrics (CloudWatch, LB)
why is monitoring down? same cluster/network? storage full? overloaded by the incident?
fix the user problem first; restore monitoring in parallel (second person)
```

**The one thing to know.** Know your **"no-dashboard" commands** by heart (topic 08 kit) — they're how you debug when the tools you depend on are part of the outage.

> 📖 Read only **Symptoms**, then list three ways to measure impact without Grafana.

---

## Symptoms

- Users and support report slowness and login failures; status page green (nobody updated it).
- Grafana: "No data"; Prometheus UI slow or unreachable; Alertmanager silent; log search times out.
- The Watchdog/heartbeat alert (topic 44) — if configured — has gone quiet, and the external heartbeat service may be paging "monitoring dead".

## Analogy

A pilot flying when the cockpit screens go dark. They switch to the backup instruments and look out of the window, and they keep flying the plane first.

## How it happens

Topic 44: monitoring shares fate with production (same cluster, network, DNS, storage), or the incident's own flood (logs, high-cardinality metrics) overloads the monitoring pipeline; or monitoring itself is the cause (e.g. an agent overloading the API server, topic 65).

### Lab setup (break it)

```bash
# With kube-prometheus-stack from topic 76 installed:
kubectl -n monitoring scale statefulset prometheus-mon-kube-prometheus-stack-prometheus --replicas=0   # name varies; check 'kubectl -n monitoring get sts'
# And break the app quietly:
kubectl -n shop set resources deploy/web --limits=cpu=20m      # heavy throttling → slow
```

## How to find it

**First — impact from outside (independent of your monitoring):**

```bash
for i in $(seq 10); do curl -s -o /dev/null -w '%{http_code} %{time_total}\n' https://app.example.com/login; done
```
```
200 4.81
200 5.02
504 10.00
```
Also: external synthetic monitor (if separate), support ticket volume, payment provider's dashboard (orders/min), CDN analytics, cloud LB metrics (CloudWatch `TargetResponseTime`, `HTTPCode_Target_5XX_Count`).

**Second — debug directly:**

```bash
kubectl get nodes; kubectl top nodes
kubectl -n shop get pods -o wide; kubectl -n shop top pods
kubectl -n shop get events --sort-by=.lastTimestamp | tail -20
kubectl -n shop logs deploy/web --since=30m | tail -50
kubectl -n shop get pod -l app=web -o jsonpath='{.items[0].spec.containers[0].resources}'
```
On a node: `kubectl debug node/<node> -it --image=busybox` or SSM; `cat /sys/fs/cgroup/.../cpu.stat` for throttling (topic 15).

**Third — why is monitoring down?** (assign to a second person)

```bash
kubectl -n monitoring get pods
kubectl -n monitoring describe pod <prometheus-pod> | tail -20
kubectl -n monitoring get pvc; kubectl -n monitoring logs <prometheus-pod> -c prometheus --tail=50
```
Common: Prometheus OOMKilled by a cardinality explosion, its disk full, scaled to zero by a bad change, or unreachable via the same broken network.

## How to fix it

### Immediate

- Fix the **user** problem with direct evidence (here: restore CPU limits / roll back the resource change).
- Update the **status page** manually based on outside measurements.
- Restore monitoring (scale back up, give it memory/disk, drop the noisy metric/log source) — without starving production.

### Permanent

- **Watchdog → external heartbeat** service so "monitoring is dead" pages someone (topic 44).
- **External synthetics** from another provider.
- Monitoring in a **separate failure domain** (separate cluster/account); cloud-native metrics as a backup view.
- Cardinality limits and log sampling so incidents don't kill observability.
- A "no-dashboard" runbook and drill.

## How to verify the fix

- Outside checks: latency and status codes normal; support tickets stop.
- Monitoring: data flowing again; backfilled view shows the incident window (if retained); Watchdog heartbeat arriving; test alert delivered.

## Prevention

Meta-monitoring, independent synthetics, monitoring change control, quarterly blind drills.

## Follow-up questions

1. **Q: Root cause?**
   A: Two problems: the app was CPU-throttled after a resource change, and the monitoring stack was down (scaled to zero / overloaded) with no heartbeat alert, so nobody was paged.
2. **Q: How did you detect it? Why not earlier?**
   A: A phone call from support. A dead-man's switch and external synthetics would have paged within minutes.
3. **Q: What did you check first?**
   A: Impact from outside, because silence from monitoring means nothing.
4. **Q: How did you stop impact quickly?**
   A: Used kubectl to find throttled pods, reverted the resource change; updated the status page manually.
5. **Q: Permanent fix?**
   A: External heartbeat for alerting, external synthetics, separate monitoring failure domain, cardinality and log controls.
6. **Q: How did you confirm?**
   A: Outside checks normal; monitoring restored; test alert delivered end to end.
7. **Q: What if kubectl also didn't work?**
   A: Use the cloud console and provider metrics, node access via SSM/serial console, and the managed Kubernetes provider's control-plane logs; prepare break-glass access in advance (topic 65).
8. **Q: How would you prevent it?**
   A: Make "monitoring is broken" itself an alert, and keep a monitoring path that shares nothing with production.
9. **Q: What would you do differently?**
   A: Split roles immediately: one person fixes users' problem, another restores monitoring.
10. **Q: What if the incident's own logs were what killed the logging system?**
    A: Drop or sample the noisiest source at the shipper (Fluent Bit filters), protect error and SLO metrics first, and add rate limits per source.
