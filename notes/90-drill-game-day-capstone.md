# 90 · Capstone: A Game Day with Several Faults

> **Level:** ★★★ advanced
>
> **You will practise:** everything at once: a **multi-fault** incident with real **roles** (IC, Ops, Comms, Scribe), changing severity, misleading signals, a mitigation that **fails**, communication under pressure, and a full **blameless postmortem** at the end. This is the closest you'll get to a real SEV1 before having one.
>
> **You need:** all of Parts 1–4.
>
> **Part 4 · On-call drills** · Previous: [89 · Monitoring is down too](89-drill-monitoring-down.md) · Next: [91 · "Tell me about an incident"](91-tell-me-about-an-incident.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **game master** (injector) runs a script of faults that unfold over 60–90 minutes. The **team** responds with roles. Real incidents rarely have one clean cause; this drill trains you to handle **overlapping** problems calmly.

```
T+0   fault 1: bad deploy → 5xx 5%
T+10  fault 2: retry storm on the DB (from the bad deploy's retry logic) → latency everywhere
T+20  twist: rollback "doesn't work" (DB still overloaded — metastable)
T+30  fault 3: monitoring dashboards stop updating (Prometheus OOM from new high-cardinality metric)
T+40  twist: an exec asks for an ETA; a customer posts on social media
T+60  recovery + resolve → 30-minute postmortem
```

**The one thing to know.** **Roles and one-change-at-a-time** matter more than technical brilliance when several things break.

**You've got it if you can:** run this game day, keep a timeline, and produce a postmortem with at least three contributing factors and specific actions.

> 📖 **That's the core.** Below: the full script, the roles, the scoring and a model postmortem.

---

## Symptoms (what the team sees first)

> 📟 **T+0:** `checkout error budget burn 10× — 5xx 5%`. Deploy history shows `checkout v3.1` 4 minutes ago.

Everything else is revealed by the game master over time.

## Analogy

A full-scale emergency exercise: fire, plus a power cut, plus a journalist at the gate. The point isn't the fire; it's whether the team coordinates.

## How it happens (the game master's script)

**Setup (lab):** the kind cluster from topic 76 with `kube-prometheus-stack`, a `checkout` Deployment, a PostgreSQL pod, a small load generator, and Prometheus alerts for 5xx and latency.

| Time | Fault injected | What the team should eventually do |
|---|---|---|
| T+0 | Deploy `checkout v3.1` with retries ×3, no backoff, and a bug returning 500 for 5% of requests | Correlate with deploy → roll back (topic 77) |
| T+10 | Slow the DB (e.g. a heavy query loop in the DB pod) so v3.1's retries amplify load | Notice DB saturation; latency on all services (topics 36, 38) |
| T+20 | After rollback, keep DB slow for 10 more minutes (metastable) | Don't flip-flop; shed load / kill heavy queries; confirm one change at a time |
| T+30 | Scale Prometheus to 0 (or OOM it) | Switch to direct tools; assign someone to restore monitoring (topic 89) |
| T+40 | Game master (as exec): "When will it be fixed? Customers are tweeting." | Comms lead answers with facts + next update time (topic 06) |
| T+50 | Stop DB load if the team hasn't found it; otherwise let them recover | Verify with user-facing signals; downgrade; resolve |

## How to find it

The team runs topic 07's script; the game master watches for:
- Did they declare and set **severity** early, and **upgrade** when latency spread?
- Did they assign **roles** (IC not typing)?
- Did they ask **"what changed?"** and **roll back** first?
- When rollback didn't fix everything, did they **form new hypotheses** instead of panicking or reverting the rollback?
- Did they **follow the request** to the DB and spot the retry amplification?
- Did they handle **monitoring loss** with direct tools?
- Did Comms keep a **cadence**?

## How to fix it

Expected path: roll back v3.1 → identify DB saturation from retries + heavy queries → cancel heavy queries / shed load → let the DB recover → readmit traffic → restore Prometheus → verify → resolve.

## How to verify the fix

- User-facing 5xx and latency back to SLO for 15 minutes.
- DB load normal; no heavy queries running.
- Monitoring restored and showing the recovery.
- IC declares resolved and posts a final update.

## Prevention (the postmortem)

Run a **30-minute blameless review** immediately. Model postmortem:

```markdown
# Game day postmortem — checkout v3.1 + DB overload (drill)
Impact (simulated): 5xx up to 9%, p99 latency 6 s, 52 minutes; dashboards blind for 20 minutes
Timeline: (from the Scribe)
Contributing factors:
 1. v3.1 retried 3× without backoff → amplified load on a slowing DB
 2. No canary analysis; 100% rollout in 2 minutes
 3. DB heavy query with no statement timeout
 4. Monitoring in the same cluster; a new high-cardinality metric caused Prometheus OOM; no heartbeat alert
 5. Comms: first status update at T+18 (target: T+10)
What went well: rollback in 6 minutes; roles assigned by T+4
What went badly: rollback was reverted by mistake at T+22 (two people acting at once)
Action items:
 - Prevent: shared retry library with backoff + jitter + budget (owner, date)
 - Prevent: canary with automated analysis for checkout (owner, date)
 - Detect: DB saturation alert; Watchdog → external heartbeat (owner, date)
 - Mitigate: statement_timeout for app roles; runbook "DB overload: shed + kill queries" (owner, date)
 - Process: "announce before you act" enforced by IC (owner, date)
```

### Scoring (game master, out of 20)

| Area | Points |
|---|---|
| Declared, severity set and updated | 3 |
| Roles assigned and respected (IC not typing) | 3 |
| "What changed?" + rollback within 10 minutes | 3 |
| Correct handling of the failed mitigation (new hypothesis, one change at a time) | 3 |
| Debugging without dashboards | 2 |
| Communication cadence and quality | 3 |
| Postmortem quality (contributing factors + specific actions) | 3 |

## Follow-up questions

1. **Q: What was the real root cause of the drill incident?**
   A: A release with retry logic lacking backoff hit a database that was slowing down, creating a retry storm; contributing factors were no canary, no DB statement timeouts, and monitoring sharing fate with production.
2. **Q: How was it detected? What was missed?**
   A: Burn-rate alert on checkout. Missed: DB saturation alerting and a heartbeat for monitoring.
3. **Q: What was checked first?**
   A: Deploy history, because the start lined up with v3.1.
4. **Q: How was impact stopped?**
   A: Rollback, then shedding load and killing heavy queries to let the DB recover.
5. **Q: Why didn't the rollback fix everything?**
   A: The system had entered a metastable overload: queued work and retries kept the DB saturated after the trigger was removed.
6. **Q: How did you confirm recovery?**
   A: User-facing SLIs normal for 15 minutes; DB load normal; monitoring back and consistent.
7. **Q: What if two people had made changes at once (as in the model postmortem)?**
   A: That's why the IC enforces "announce before you act" — conflicting changes hide what works and can undo mitigations.
8. **Q: How would you prevent this class of incident?**
   A: Resilient client defaults (backoff, budgets), canaries, DB timeouts, independent monitoring.
9. **Q: What would you do differently in the next game day?**
   A: Rotate roles, add a communication-only participant, and include a "rollback is impossible" twist.
10. **Q: How often should a team run game days?**
    A: Regularly enough that everyone on the rota has practised IC and Ops roles recently — many teams aim for quarterly, plus smaller drills more often.
