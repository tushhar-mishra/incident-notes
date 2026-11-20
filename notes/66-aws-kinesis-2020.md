# 66 · AWS Kinesis, November 2020: Too Many Threads

> **Level:** ★★ intermediate
>
> **Pattern:** **Adding capacity** (a normally safe change) pushes every server past an **operating-system thread limit**, because each server opens a thread **per other server** → shard maps fail to build → Kinesis can't route → dependent services (**Cognito, CloudWatch, Lambda, EventBridge**) fail → the **status page** can't be updated. **Families:** capacity (resource limits, topic 14) · dependency (topic 39) · hidden O(N²) scaling.
>
> **You will learn:** why "each server talks to every other server" means resources grow with fleet size, how a small capacity addition crossed an OS limit on **every** server at once, why recovery took **most of a day** (restart only a few hundred servers per hour), how Kinesis problems cascaded into other AWS services, and AWS's fixes (bigger servers → fewer servers, thread alarms, **cellularization**).
>
> **You need:** Part 1, topics 14, 38, 39.
>
> **Part 3 · Real outages · Capacity and overload** · Previous: [65 · OpenAI 2024](65-openai-kubernetes-2024.md) · Next: [67 · Slack 2021](67-slack-2021.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Each Kinesis **front-end** server keeps a thread for **every other** front-end server (for sharing membership and shard information). Add servers, and **every** server needs more threads. On 25 November 2020, a "relatively small" capacity addition pushed **all** servers over the OS's **maximum number of threads**. Their shard-map caches couldn't be built, so they couldn't route requests to the back end.

```
threads per server ≈ (number of front-end servers) + other work
2:44–3:47 AM PST: add capacity → every server exceeds OS max threads
   → shard-map cache construction fails → front ends can't route → Kinesis errors (alarms 5:15 AM)
   → Cognito, CloudWatch (metrics/logs), Lambda, EventBridge, ECS/EKS affected
   → Service Health Dashboard posting tool uses Cognito → can't update status
9:39 AM root cause confirmed → remove new capacity, restart fleet slowly (few hundred servers/hour)
10:23 PM PST: Kinesis fully back to normal
```

**Picture it.** A club where every member must have a personal phone line to every other member. With 100 members that's manageable; every new member means **every** existing member needs one more line. One day the building's phone exchange hits its maximum lines and nobody can call anyone.

**The one thing to know.** Watch for designs where **per-node resources grow with cluster size** (full-mesh connections, gossip, per-peer threads). They fail on **scale-up**, at the worst time, and on **all nodes together**.

**You've got it if you can:** explain why adding capacity caused the outage, and why the restart was so slow.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 25 November 2020, AWS added a relatively small amount of capacity to the Kinesis Data Streams front-end fleet in us-east-1. Because each front-end server creates an OS thread for every other front-end server, the addition caused all servers to exceed the maximum number of threads allowed by an OS configuration; shard-map caches failed to build and the fleet couldn't route requests. Kinesis and dependent services (Cognito, CloudWatch, Lambda, EventBridge and others) were impaired until AWS removed the new capacity and slowly restarted the fleet; Kinesis returned to normal at 10:23 PM PST.

## Impact

| Item | Detail (from AWS's summary) |
|---|---|
| Kinesis | Errors putting and getting records from 5:15 AM; fully normal 10:23 PM PST |
| Cognito | Elevated API failures and latency (User Pools, Identity Pools); normal by 2:18 PM |
| CloudWatch | Errors/latency for PutMetricData and PutLogEvents from 5:15 AM; alarms to INSUFFICIENT_DATA; fully recovered 10:31 PM |
| Auto Scaling | Reactive policies relying on CloudWatch delayed until 5:47 PM |
| Lambda | Memory contention on invocation hosts from 6:15 AM (buffered metrics); mitigated 10:36 AM |
| EventBridge / ECS / EKS | API errors and delays from 5:15 AM; most resolved by 4:15 PM |
| Status page | Couldn't be updated early on (posting tool uses Cognito) |
| Customers / cost | **Not publicly disclosed** |

## Timeline (PST, 25 November 2020)

| Time | Event |
|---|---|
| 2:44 → 3:47 AM | Capacity added to the front-end fleet |
| 5:15 AM | First alarms for errors putting and getting Kinesis records |
| 7:01 AM | Cognito errors rise significantly |
| 7:51 AM | Root cause narrowed to a couple of candidates; full front-end restart judged necessary |
| 9:39 AM | Root cause confirmed (OS thread limit, not memory pressure) |
| 10:07 AM | First group of restarted servers taking traffic |
| Noon → | Error rates steadily dropping |
| 2:18 PM | Cognito normal |
| 10:23 PM | Kinesis fully back to normal |

## How it happened

```
Front-end fleet (many thousands of servers)
  each server:  thread → server 2, thread → server 3, ..., thread → server N
                + builds a "shard-map" cache (membership, shard ownership) using those connections
  add capacity (N grows) → threads per server grow on EVERY server
  → all exceed OS max threads → shard-map build fails → useless shard maps → can't route to back ends
```

AWS: the new capacity was "not root cause", but it "caused all of the servers in the fleet to exceed the maximum number of threads allowed by an operating system configuration."

## Why it spread

- **All servers hit the limit together** — the extra threads were needed everywhere at once.
- **Dependencies:** Cognito (via a buffering bug that let webservers block on Kinesis backlogs), CloudWatch, Lambda, EventBridge, and through them ECS/EKS and Auto Scaling.
- **Status page dependency:** the tool for posting to the Service Health Dashboard used Cognito; AWS used a backup tool with minimal dependencies, which caused delays because it was more manual and less familiar.

## How they found it

- Alarms at 5:15 AM (errors on put/get).
- Diagnosis was hard because errors appeared "in all aspects of the various calls", making side effects hard to separate from the cause. Memory pressure was a candidate before the thread limit was confirmed at 9:39 AM.

## What they did to fix it

1. **Removed the new capacity** (they chose not to raise the OS thread limit without further testing).
2. **Restarted the whole front-end fleet** carefully: shard-map building competes with request processing for resources, so servers had to be added back slowly — "a few hundred per hour".
3. Worked on letting servers bootstrap shard maps from the authoritative metadata store instead of from peers.
4. Mitigated dependent services (Cognito change so webservers survive Kinesis errors; Lambda memory contention fix).

## Permanent fix (as listed by AWS)

| Change | Pattern it addresses |
|---|---|
| Move to **larger CPU and memory servers** → fewer servers → fewer threads | O(N) per-server resource |
| **Fine-grained alarming for thread consumption** | Visibility of the limit |
| Finish testing an **increase of the OS thread limit** | Headroom |
| Move the front-end **cache to a dedicated fleet**; move large services like CloudWatch to a **separate, partitioned front-end fleet** | Cold start and blast radius |
| Accelerate **cellularization** of the front-end fleet | Blast radius |
| Cognito: webservers sustain Kinesis API errors without exhausting buffers | Dependency isolation |
| CloudWatch: persist 3 hours of metric data locally | Ride out dependency failures |
| Support trained on the backup status-posting tool | Communications |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Full-mesh or gossip clusters (Consul, Cassandra, Elasticsearch, Redis Cluster, service meshes) | Know per-node cost vs cluster size; load-test scale-up, not just steady state |
| Linux limits: `kernel.threads-max`, `pid_max`, `ulimit -u`, systemd `TasksMax`, Kubernetes PID limits | Monitor threads/PIDs per process and node; alert near limits (topics 14, 33; [Linux 033](../../linux-mastery-rhel/notes/033-processes-and-signals.md)) |
| Scaling out as a "safe" mitigation | Scaling out can trigger limits like this one; watch resource usage on **existing** nodes during scale-up |
| Status/incident tools that depend on production services | Backup tools, practised regularly |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A design where every front-end server holds a thread per other server, so thread usage grows with fleet size, with no alarm on thread consumption. Adding capacity was the trigger; it pushed every server over the OS thread limit.

2. **Q: How was it detected, and why did diagnosis take hours?**
   A: Alarms on put/get errors at 5:15 AM. Errors were everywhere, making side effects hard to separate from the cause; memory pressure was considered before threads were confirmed.

3. **Q: What would you check first?**
   A: What changed (the capacity addition), then resource limits on existing servers: threads per process vs limit, PIDs, file descriptors.

4. **Q: How did they stop the impact?**
   A: Removed the new capacity and restarted the fleet slowly so shard-map building didn't starve request processing.

5. **Q: Why couldn't they just raise the thread limit?**
   A: It hadn't been tested; changing a fleet-wide OS limit under pressure could cause new problems. They chose the known-safe path.

6. **Q: What changed permanently?**
   A: Bigger and fewer servers, thread alarms, tested higher limits, dedicated cache fleet, cellularization, and dependency hardening in Cognito and CloudWatch.

7. **Q: How would you confirm the fix?**
   A: Load-test scale-up to well beyond current size and watch thread counts on existing servers; alarms trigger well before limits.

8. **Q: How would you prevent this class of incident?**
   A: Avoid O(N) per-node resources; cell-based designs keep each cell small; alarm on every OS limit.

9. **Q: What would you do differently?**
   A: Treat capacity additions as changes with canaries: add a few servers, watch every limit on the existing fleet, then add more.

10. **Q: What if this happened to a cluster you run (e.g. a gossip-based database) during a traffic spike?**
    A: Scaling out would make it worse. Scale **up** instead (bigger nodes), shed load, and plan cluster size limits in advance.

## Source

- AWS, "Summary of the Amazon Kinesis Event in the Northern Virginia (US-EAST-1) Region" (25 November 2020): <https://aws.amazon.com/message/11201/>
