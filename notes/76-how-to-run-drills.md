# 76 · How to Run Drills and Game Days

> **Level:** ★ foundation
>
> **You will learn:** how to turn every scenario in Part 4 into **hands-on practice**: setting up a safe **lab** (kind cluster, Docker, a small VM), the roles for a **solo drill** vs a **team game day**, the **drill loop** (inject → detect → triage → mitigate → verify → write up), how to **score** yourself, how to run **tabletop** drills when you can't break anything, and how to graduate to **chaos engineering** in staging and production.
>
> **You need:** Parts 1–3. Helpful: [SD 44 · Chaos engineering and game days](../../System-Design/notes/44-chaos-engineering-and-game-days.md), [K8s 11 · Lab setup](../../CKA-Certification-Course-2025/notes/11-lab-setup-kind-and-minikube.md), [Docker 52 · Production scenarios](../../Docker/notes/52-production-scenarios.md).
>
> **Part 4 · On-call drills** · Previous: [75 · CircleCI 2023](75-circleci-2023.md) · Next: [77 · 5xx after a deploy](77-drill-5xx-after-deploy.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** You don't learn incident response by reading. You learn it by **doing it under mild pressure**, many times, somewhere safe. A drill is a **planned, fake incident**: someone (or a script) breaks something, and you respond as if paged.

```
PREPARE  lab + monitoring + the fault script + a timer
INJECT   break it (without looking, if a partner does it)
RESPOND  ack → impact → what changed → mitigate → verify   (topic 07 script)
REVIEW   timeline, what went well/badly, score, fix the runbook
```

**Picture it.** Fire drills. Nobody learns to evacuate from a manual. Practising on a calm day means that on the real day your hands know what to do.

**The one thing to know.** **Write down your timeline during the drill** (even solo). The review is where the learning happens.

**Try it now:** set up the lab below and run drill 77 with a 30-minute timer.

**You've got it if you can:** run a drill end to end and produce a 1-page mini-postmortem.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Type | What | When |
|---|---|---|
| **Solo drill** | You break something in your lab with a script, then fix it | Learning a pattern |
| **Pair drill** | A partner breaks it secretly; you respond | Realistic diagnosis |
| **Tabletop** | Talk through a scenario with no real systems: "the DB is down, what do you do?" | Teams, process, communication |
| **Game day** | A planned team exercise on staging (or carefully in production) with real roles | Testing runbooks, tooling and coordination |
| **Chaos engineering** | Automated, continuous, controlled fault injection with a hypothesis | Mature systems |

---

## Why it exists

- Real incidents are rare and stressful; drills make them familiar.
- Drills find **broken runbooks, missing alerts and missing access** before a real incident does (e.g. "the break-glass account's password expired").
- Interviewers ask "how do you prepare for incidents?" — drills are the senior answer.

---

## Analogy

**Fire drills** (Core). A **game day** is a full emergency exercise with fire engines, actors playing casualties and observers taking notes.

---

## How it works

### The lab (free, local)

```bash
# Kubernetes in Docker (from your K8s notes, topic 11)
kind create cluster --name drills
kubectl create namespace shop

# Monitoring (optional but recommended): kube-prometheus-stack via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install mon prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# A simple app to break
kubectl -n shop create deployment web --image=nginx:1.27 --replicas=3
kubectl -n shop expose deployment web --port=80
kubectl -n shop run probe --image=busybox:1.36 --restart=Never -- sh -c \
  'while true; do wget -q -O /dev/null -T 2 http://web && echo ok || echo FAIL; sleep 1; done'
kubectl -n shop logs -f probe      # your "synthetic check"
```

Plus a Linux VM or container for host drills (disk, FDs, clock), and a free-tier cloud account **with a budget alert** for cloud drills (or do those as tabletops).

### Roles

| Solo | Pair / team |
|---|---|
| You are IC, Ops and Scribe; keep a text file timeline | **Injector** (breaks things, plays "users" and "managers"), **IC**, **Ops**, **Scribe**, **Observer** (scores) |

### The drill loop

1. **Set the scene.** Read only the drill's **Alert** and **Symptoms**. Start a timer.
2. **Respond** using topic 07's script. Write each step with a timestamp.
3. **Stop at "mitigated and verified"** (the user-facing check is green again).
4. **Review** (10 minutes): compare with the drill's answer; write a mini-postmortem:

```
Drill: 79 latency, no deploy          Date: ____   Time to mitigate: 23 min
What I checked, in order: ...
First correct hypothesis at: 14 min
What slowed me down: didn't know the query to see pool usage
Fix to my runbook/notes: add pool metric query to cheat sheet
Score (below): 7/10
```

### Score yourself

| Criterion | Points |
|---|---|
| Acknowledged and stated impact (who, how bad, since when) within 5 min | 2 |
| Asked "what changed?" and checked deploys/config/infra/traffic/deps | 2 |
| Chose a safe, fast mitigation before deep root-causing | 2 |
| Verified with the **user-facing** signal, not component health | 2 |
| Wrote a timeline + at least one prevention action | 2 |

### Tabletop format (for teams, 45 minutes)

1. Facilitator reads the scenario and reveals new information every few minutes ("the rollback didn't help", "the CEO is asking", "Slack is down").
2. Participants say what they'd do and who does it.
3. Facilitator notes gaps: missing access, unknown owners, unclear runbooks.
4. End with three action items.

### From drills to chaos engineering

Start in the lab → staging game days → small, controlled experiments in production with: a **hypothesis** ("if one AZ's pods die, error rate stays < 0.5%"), a **blast radius limit**, a **stop button**, and people watching. Tools: Chaos Mesh, LitmusChaos, AWS Fault Injection Service, Gremlin.

---

## Failure modes (of drills)

| Failure | Fix |
|---|---|
| Drills too easy / always the same | Rotate scenarios; let a partner choose; add twists (topic 90) |
| Breaking something real by accident | Labs and staging first; production only with stop buttons and approval |
| No follow-up | Every drill produces at least one runbook/alert fix |
| Only engineers practise | Include comms and IC roles; tabletop with managers |

---

## Where you'll use it

- Every drill in 77–90.
- Real incidents in Part 3 that mention drills: Facebook's **storm** drills helped recovery (topic 59); OpenAI committed to **fault injection** and rapid cluster replacement exercises (topic 65); Atlassian to large-scale **DR exercises** (topic 47).

---

## Quick recap

- Drill loop: prepare → inject → respond → review.
- Keep a timestamped timeline; score yourself on five criteria.
- Tabletops for process and communication; game days for tooling; chaos engineering for continuous confidence.
- Every drill should change something (runbook, alert, access).

### Interview questions

1. **Q: How do you prepare a team for incidents?**
   A: Runbooks linked from alerts, regular drills (tabletops for process, game days for tooling), rotating roles so everyone practises IC and comms, and fixing whatever each drill exposes. Mature teams add controlled chaos experiments with hypotheses and stop buttons.

2. **Q: What's the difference between a game day and chaos engineering?**
   A: A game day is a planned event with people practising response to an injected failure. Chaos engineering is a continuous practice of hypothesis-driven fault injection, often automated, to build confidence that the system tolerates failures.

3. **Q: How do you run chaos experiments safely in production?**
   A: Start small with a clear hypothesis and steady-state metric, limit the blast radius (one pod, one AZ, a percentage of traffic), have an instant stop, run during staffed hours, and expand only after success.
