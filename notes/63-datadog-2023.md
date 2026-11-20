# 63 · Datadog, March 2023: The Update Nobody Scheduled

> **Level:** ★★ intermediate
>
> **Pattern:** A **forgotten automatic update channel** in the base OS image + the **same default time window** everywhere → a systemd security update restarts `systemd-networkd` on many Kubernetes nodes in **every region and cloud at once** → it **flushes routing rules** installed by the CNI (Cilium) → nodes go offline. **Families:** change (dependency/OS update, topic 25) · platform/Kubernetes · recovery at scale.
>
> **You will learn:** how Datadog's own careful **region-by-region** rollout process was bypassed by an Ubuntu **unattended-upgrades** channel nobody used, why the bug only appeared when `systemd-networkd` **restarted** (not on fresh or rebooted nodes), why every region failed **within the same hour** (06:00–07:00 UTC default window), why recovery took more than a day (**tens of thousands of nodes**, different cloud behaviours, cloud rate limits), and the three fixes.
>
> **You need:** Part 1, topics 21, 25, 28.
>
> **Part 3 · Real outages · Platform, OS and Kubernetes** · Previous: [62 · AWS DynamoDB DNS 2025](62-aws-dynamodb-dns-2025.md) · Next: [64 · Heroku 2025](64-heroku-2025.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Datadog rolls out changes "region-by-region, cluster-by-cluster, node-by-node". But its Kubernetes node image still had Ubuntu's **legacy security update channel** enabled, which runs automatically every day between **06:00 and 07:00 UTC** by default. On 8 March 2023 it installed a **systemd** security update. Restarting `systemd-networkd` on Ubuntu 22.04 (systemd v249) made it **flush IP routing rules it didn't create** — including those installed by **Cilium**, the Kubernetes network plugin. Nodes lost networking. Because every region used the same image and the same time window, all regions failed together.

```
Base OS image: unattended security upgrades ON (legacy, forgotten)   default window 06:00–07:00 UTC
Mar 7: Ubuntu publishes systemd security update
Mar 8 06:00 UTC: nodes in US1, EU1, US3, US4, US5 install it → systemd-networkd restarts
   → networkd flushes "foreign" IP rules → Cilium's rules gone → nodes offline
06:03 outage starts (all services, all regions) → detected by internal monitoring in ~3 minutes
Mar 9 08:58: all services operational · Mar 10 06:25: fully resolved (historical data backfilled)
```

**Picture it.** A hospital carefully tests every new medicine on one ward before others. But every ward's medicine cabinet also has an old automatic delivery that drops in a new pill at 6 AM daily. Nobody remembers it. One morning, the pill reacts badly with a drug every patient takes — on every ward in every hospital at once.

**The one thing to know.** **Your change process is only as good as the changes it controls.** Find every **automatic** change mechanism (OS updates, agents, managed upgrades) and bring it under your staged rollout — or turn it off.

**You've got it if you can:** explain why the bug didn't appear on new or rebooted nodes, and why "multi-region, multi-cloud" didn't protect Datadog.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 8 March 2023, a systemd security update was applied automatically to many of Datadog's Ubuntu 22.04 Kubernetes nodes through a legacy update channel left enabled in the base OS image. Restarting `systemd-networkd` caused it to delete routing rules managed by the Cilium CNI, taking nodes offline. Because the update ran in the same default window (06:00–07:00 UTC) everywhere, all of Datadog's regions across several cloud providers were affected at once. Services were operational again on 9 March and data was fully backfilled by 10 March.

## Impact

| Item | Detail (from Datadog's postmortem and deep dive) |
|---|---|
| Regions | **US1, EU1, US3, US4, US5** — "across all services" |
| Duration | Outage from 06:03 UTC on 8 March; all services operational 08:58 UTC on 9 March; fully resolved 06:25 UTC on 10 March (after historical data backfill) |
| Scale of recovery | "tens of thousands of nodes" replaced or restarted |
| Response | An emergency operation center of 10 senior engineering leaders, about 70 local incident commanders and 450–750 responders, in four shifts; about 200 status updates across four regions |
| Customers / cost | **Not publicly disclosed** |

## Timeline (UTC)

| Time | Event |
|---|---|
| 7 Mar | Ubuntu releases systemd package 249.11-0ubuntu3.7 |
| 8 Mar 06:00 | First faulty upgrade applied (unattended upgrades run 06:00–07:00 with a randomized delay) |
| 06:03 | Outage starts; internal monitoring alerts about three minutes after the first faulty upgrade |
| 06:31 | High-severity incident declared; first status page update |
| 07:23 | Status update: impact to data ingestion and monitors |
| 09:13 | Initial signs of recovery (web access restored) |
| 16:44 | First major service declared operational |
| 9 Mar 08:58 | All services operational in all regions |
| 10 Mar 06:25 | Fully resolved after backfilling historical data |

## How it happened

```
Node (Ubuntu 22.04, systemd v249) running Cilium
  boot: systemd-networkd starts FIRST → then Cilium installs its IP rules → OK
  (so fresh nodes and rebooted nodes are fine — the bug was invisible)

  06:0x: unattended-upgrades installs new systemd → package update RESTARTS systemd-networkd
         → on start, networkd "flushes all IP rules it does not know about"
         → Cilium's routing policy rules deleted → node loses network connectivity
         → Kubernetes sees node offline → cloud/K8s replaces or marks it unhealthy
```

Key facts from Datadog:
- The behaviour "only manifests when systemd-networkd starts", which in their infrastructure normally happened only when a host was created — so it had never been seen.
- Ubuntu 20.04 nodes (systemd v245) got a similar patch but don't flush IP rules on restart, so they weren't affected; 22.04 was **over 90%** of the fleet.
- Datadog: "We had never disabled unattended upgrades", even though "we do not rely on unattended-upgrades to update our nodes".

## Why it spread

- **Same image + same default schedule** → every region updated in the same hour, "even though the regions have no direct network connection or coupling."
- **Multi-cloud didn't help**: the cause was in the OS image used on all clouds.
- **Recovery at enormous scale**:
  - Capacity had to be restored first, and scaled **higher than normal** to process real-time and backlog data together.
  - Cloud providers behaved differently: some marked nodes unhealthy without retiring them (so a reboot recovered them, keeping local data); others **immediately replaced** unhealthy nodes, **losing local data** on them. Replacing tens of thousands of nodes at once "tested the cloud provider's regional rate limits in various ways, none of which was obvious ex ante."

## How they found it

**Internal monitoring** alerted about three minutes after the first faulty upgrade; a high-severity incident was declared at 06:31. (How they pinpointed the systemd update is covered in their deep dive; the main postmortem focuses on scope and recovery.)

## What they did to fix it

1. Recover **compute capacity** in each region (higher than normal), then recover services in parallel.
2. Prioritise **real-time** telemetry first, then process the historical data that had been ingested but not processed.
3. Where nodes were still running without networking, restart them; where clouds replaced them, rebuild state.
4. Communicate continuously (about 200 status updates) with support in contact with customers around the clock.

## Permanent fix (as listed by Datadog)

| Change | Pattern it addresses |
|---|---|
| **Disabled the legacy security update channel** across all affected regions (redundant with their change management tooling) | Uncontrolled automatic change |
| Changed **systemd-networkd configuration** so it leaves the routing table unchanged on restart | Component that destroys others' state |
| **Audited infrastructure** for similar legacy update channels | Find the class, not just the instance |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Node images (EKS AMIs, Packer images) with `unattended-upgrades` / `dnf-automatic` enabled | Disable in the image; patch by rolling new images per node group (topic 25) |
| CNI (Calico, Cilium, AWS VPC CNI) relying on rules that other services might remove | Test `systemctl restart systemd-networkd` (and NetworkManager, firewalld reloads) on a node in staging ([Linux 060 · NetworkManager](../../linux-mastery-rhel/notes/060-networkmanager-and-nmcli.md), [K8s 29 · CNI](../../CKA-Certification-Course-2025/notes/29-network-model-and-cni.md)) |
| Same cron/timer schedule on every host | Randomise and stagger; different windows per region |
| Assuming "multi-region" = independent | List what all regions share: images, configs, pipelines, vendors, schedules |
| Autoscalers that replace unhealthy nodes immediately (losing local disks) | Know your cloud's behaviour; keep stateful data on network volumes or replicated |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: An automatic OS security update channel left enabled in the base image, outside the staged change process, combined with a systemd-networkd behaviour that flushed Cilium's routing rules on restart. The systemd update was the trigger.

2. **Q: How was it detected, and why not earlier?**
   A: Internal monitoring in about three minutes. The behaviour couldn't be seen earlier because it only happened when systemd-networkd restarted on a running node, never on fresh or rebooted nodes.

3. **Q: What would you check first?**
   A: What all failing nodes share and what ran on them at 06:00: package logs (`/var/log/apt/history.log`), `journalctl` for systemd restarts, `ip rule` output vs a healthy node.

4. **Q: How did they stop the impact?**
   A: The update had already run; recovery meant restoring capacity (restart/replace nodes), then services and data, prioritising real-time data.

5. **Q: What changed permanently?**
   A: The legacy update channel disabled, systemd-networkd configured to leave routes alone on restart, and an audit for similar channels.

6. **Q: How would you confirm the fix?**
   A: Check no node has the update timer enabled; restart systemd-networkd on a test node and confirm CNI rules survive.

7. **Q: Why did recovery take more than a day?**
   A: Tens of thousands of nodes across clouds, some replaced with loss of local data, cloud rate limits on mass replacement, and a large backlog to process after real-time data was restored.

8. **Q: How would you prevent this class of incident?**
   A: Inventory every automatic change mechanism, put each under staged rollout, randomise schedules, and test restarts of system services on nodes running your CNI.

9. **Q: What would you do differently?**
   A: Treat the OS image as part of the platform's change surface, with the same review as Kubernetes manifests.

10. **Q: What if this happened at 10× the fleet size?**
    A: Cloud rate limits and quotas on node replacement would bite harder (topic 35). Pre-agreed quota increases and a recovery plan that restarts nodes instead of replacing them help.

## Source

- Datadog, "2023-03-08 Incident: Infrastructure connectivity issue affecting multiple regions": <https://www.datadoghq.com/blog/2023-03-08-multiregion-infrastructure-connectivity-issue/>
- Datadog Engineering, "A deep dive into the platform-level impact of the 2023-03-08 incident": <https://www.datadoghq.com/blog/engineering/2023-03-08-deep-dive-into-platform-level-impact/>
