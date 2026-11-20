# 09 · Runbooks and Playbooks

> **Level:** ★ foundation
>
> **You will learn:** what a **runbook** and a **playbook** are, the **sections** a good runbook needs, a full **example runbook**, how to keep runbooks **alive** (not out of date), how to move from **manual → scripted → automated** runbooks, and the dangers of runbooks (copy-pasting destructive commands, the S3 2017 lesson).
>
> **You need:** topics 01–08.
>
> **Part 1 · Incident basics** · Previous: [08 · Debugging under pressure](08-debugging-under-pressure.md) · Next: [10 · Incident metrics](10-incident-metrics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **runbook** is a short document, linked from an alert, that tells the on-call engineer **what the alert means, how to check it, and how to fix it**. It turns a 3 AM panic into following steps.

```
Alert: "CheckoutDBConnectionsHigh"  →  runbook link  →
  1. What it means      2. Impact      3. How to check (commands)
  4. How to mitigate    5. When to escalate, and to whom
```

A **playbook** is the bigger version: how to handle a **type** of incident (e.g. "region failover playbook", "security breach playbook").

**Picture it.** The laminated emergency card in an airplane seat pocket. Nobody reads it for fun; when you need it, it's short, clear, and in the right place.

**The one thing to know.** A runbook that's **wrong** is worse than none. Test runbooks in drills, and update them after every incident that used them.

**Try it now:** pick one alert that has paged you, and write its runbook using the template below.

**You've got it if you can:** write a runbook that a new team member could follow at 3 AM without calling you.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Term | Scope | Example |
|---|---|---|
| **Runbook** | One alert or one task | "Postgres replication lag high", "Rotate the TLS cert for api.example.com" |
| **Playbook** | A class of incident; coordinates many runbooks and people | "Regional failover", "Data breach response", "DDoS response" |
| **Automated runbook** | The steps as a script or a workflow, triggered by a button or the alert itself | A Rundeck/Ansible/Argo Workflow job that clears a full disk safely |

### Runbook template

```markdown
# Runbook: <AlertName>
Owner: <team>   Last tested: <date>   Severity: page / ticket

## What it means
One or two sentences in plain words.

## User impact
What users see if this is real. How bad. How fast it gets worse.

## Check (is it real? how bad?)
1. Dashboard: <link>
2. Command: <command>  →  what "good" and "bad" output look like

## Mitigate
Option A (most common): <steps>   ⚠️ <risks>
Option B: <steps>

## Verify
Which metric should return to normal, and how fast.

## Escalate
If not fixed in 15 min, or if <condition>: page <team> via <how>.

## Background
Links: architecture diagram, past incidents, postmortems.
```

---

## Why it exists

- **Speed:** the on-call doesn't need to rediscover the fix every time. Lower MTTR (topic 10).
- **Spreading knowledge:** the expert who wrote it can sleep; anyone on the rota can follow it.
- **Safety:** reviewed commands are safer than commands typed from memory at 3 AM.
- **Path to automation:** a good runbook is a script waiting to be written.

---

## Analogy

**A pilot's emergency checklist.** For "engine fire", pilots don't improvise. They run a short, tested checklist in a fixed order. The checklist was written calmly, on the ground, by people who studied every previous engine fire.

---

## How it works

### Example: a complete runbook

```markdown
# Runbook: NodeDiskWillFillIn4Hours
Owner: platform-team   Last tested: 2026-10-02 (game day)   Severity: page

## What it means
Disk on a Kubernetes node will be full within 4 hours at the current rate.
When full, kubelet evicts pods (DiskPressure) and containers fail to write logs.

## User impact
None yet. If ignored: pods on this node get evicted; if many nodes, capacity loss.

## Check
1. Dashboard: Grafana → "Node disk" → filter by node.
2. On the node (via `kubectl debug node/<node> -it --image=busybox` or SSH):
   df -h /var/lib/containerd /var/log        # which filesystem
   du -xh --max-depth=1 /var/log | sort -h    # biggest dirs
   Good: <80% used. Bad: >90% and climbing.

## Mitigate
A. Big container logs: find the pod writing them; ask owner to reduce logging;
   `crictl rmi --prune` removes unused images.
B. If one pod is the cause and it's misbehaving: `kubectl delete pod <pod>`
   (its log file goes away with it). ⚠️ Check it's not a single-replica stateful pod.
C. If many nodes are filling: escalate — likely a cluster-wide logging change.

## Verify
`df -h` below 80% and the 4-hour prediction alert clears within 15 min.

## Escalate
Not resolved in 30 min or >3 nodes affected: page platform secondary.

## Background
Topic 13 (disk full). Postmortem INC-1182 (2026-06, debug logging left on).
```

### Keeping runbooks alive

| Practice | Why |
|---|---|
| **Link from the alert** (annotation `runbook_url`) | It's found in seconds |
| **"Last tested" date** at the top | Shows if it's stale |
| **Update during the postmortem** of any incident that used it | Each use improves it |
| **Run it in a drill** each quarter | Finds broken steps safely (topic 76) |
| **Store it next to the code** (Git, reviewed via PR) | Changes are reviewed and versioned |
| **New team members test it** | They spot the steps experts skip in their heads |

### From manual to automated

```
Level 0  Knowledge in someone's head       ("ask Sam")
Level 1  Written runbook                    (steps + commands)
Level 2  Scripted runbook                   (one reviewed script, run by a human)
Level 3  Button / ChatOps                   (/runbook clear-disk node-7 in chat, with audit log)
Level 4  Auto-remediation                   (the alert triggers the fix; a human is told)
```

Go up a level only when the steps are stable and safe. **Auto-remediation can make incidents worse** if it acts on bad data (topics 62 and 74 are examples of automation acting on wrong inputs).

### Runbook safety

The AWS S3 outage of 2017 (topic 45) began when an engineer ran an established playbook command with **one input entered incorrectly**, which removed many more servers than intended. AWS's fix was to make the **tool** safer (remove capacity slowly, refuse to go below a minimum). Lessons for your runbooks:

- Commands that remove, delete or restart should have **limits** and **confirmations** built in.
- Prefer **scripts with checks** over raw commands to paste.
- Use **dry-run** flags where the tool has them (`terraform plan`, `kubectl --dry-run=server`, `ansible --check`).
- Show the target before acting: "This will drain 3 nodes: a, b, c. Continue? [y/N]".

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **Stale runbook** | Commands reference a cluster that no longer exists | Last-tested date; update after incidents; quarterly drills |
| **No runbook** | On-call wakes the expert every time | Rule: no paging alert without a runbook |
| **Too long** | 12 pages; nobody reads it at 3 AM | Short "do this" first; background at the end |
| **Dangerous copy-paste** | A destructive command without limits runs on the wrong target | Scripts with guards, dry runs, confirmations |
| **Automation without brakes** | Auto-remediation loops and makes it worse | Rate limits, kill switch, human notification |

---

## Where you'll use it

- **Ansible:** scripted runbooks are often Ansible playbooks run with `--check` first ([Ansible 20 · check, diff, limit](../../Ansible/notes/20-check-diff-limit.md), [Ansible 65 · Production scenarios](../../Ansible/notes/65-production-scenarios.md)).
- **Kubernetes:** `kubectl debug`, drain/cordon runbooks ([K8s 83](../../CKA-Certification-Course-2025/notes/83-data-plane-troubleshooting.md)).
- **Jenkins/GitHub Actions:** a "runbook job" with parameters and approvals is a common Level 3 ([Jenkins 20 · Build parameters](../../Jenkins/notes/20-build-parameters.md), [GHA 28 · Environments](../../GitHub-Actions-Basics-To-Production/notes/28-environments.md)).

---

## Quick recap

- Runbook = one alert or task; playbook = a whole incident type.
- Sections: meaning, impact, check, mitigate, verify, escalate, background.
- Link from the alert; date it; test it in drills; update after incidents.
- Automate gradually; build limits and confirmations into anything destructive.

### Interview questions

1. **Q: What makes a good runbook?**
   A: It's linked from the alert, short, and ordered for action: what the alert means, user impact, how to check if it's real (with commands and what good/bad output looks like), how to mitigate (with risks), how to verify, and when and how to escalate. It has an owner and a last-tested date, and it's kept in version control.

2. **Q: How do you stop runbooks going stale?**
   A: Require a runbook link for every paging alert, put a last-tested date on each, run them in regular drills, update them in every postmortem where they were used, and have new team members walk through them.

3. **Q: Should you automate remediation?**
   A: For stable, well-understood, safe fixes, yes, gradually: script first, then a button with an audit log, then automatic with notification. Add safety limits (rate limits, maximum scope, kill switch), because automation acting on bad data can turn a small problem into a big one.
