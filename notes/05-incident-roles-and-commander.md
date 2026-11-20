# 05 · Incident Roles and the Incident Commander

> **Level:** ★ foundation
>
> **You will learn:** the standard incident roles (**Incident Commander, Ops/Tech lead, Communications lead, Scribe**, plus **subject-matter experts**), what each one does and **does not** do, why the IC **doesn't type commands**, how roles **scale** from a 1-person SEV3 to a 50-person SEV1, how to **hand over** command, and where this comes from (the **Incident Command System** used by firefighters).
>
> **You need:** topics 01–04.
>
> **Part 1 · Incident basics** · Previous: [04 · On-call and alerting](04-on-call-and-alerting.md) · Next: [06 · Incident communication](06-incident-communication.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** In a big incident, give each person **one clear job**. The most important job is the **Incident Commander (IC)**: the person who **coordinates and decides**, but does not do the hands-on fixing.

```
                    ┌──────────────────────────┐
                    │  INCIDENT COMMANDER (IC) │  decides, coordinates, keeps the big picture
                    └────────────┬─────────────┘
          ┌──────────────────────┼──────────────────────┐
          ▼                      ▼                      ▼
   OPS / TECH LEAD         COMMS LEAD               SCRIBE
   investigates and        updates stakeholders,    writes the timeline
   makes changes           status page, support     (who did what, when)
          │
   subject-matter experts (database, network, payments…)
```

**Picture it.** An orchestra conductor doesn't play an instrument. If the conductor picks up a violin, nobody is keeping time.

**The one thing to know.** If the IC starts debugging, **nobody is coordinating**. The IC's hands stay off the keyboard.

**Try it now:** for your team, name who would be IC, Ops lead, Comms lead and Scribe if the main service went down right now.

**You've got it if you can:** describe what each role does in the first 10 minutes of a SEV1.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

| Role | Does | Does **not** |
|---|---|---|
| **Incident Commander (IC)** | Declares severity; assigns roles; asks "what's the impact? what changed? what are we trying next?"; makes decisions (e.g. "roll back now"); keeps the update cadence; decides when it's resolved | Debug, type commands, read logs deep-dive |
| **Ops lead / Tech lead** | Leads the hands-on investigation; proposes and executes mitigations; coordinates the engineers doing the work | Talk to customers or executives |
| **Communications lead (Comms)** | Writes internal updates and the status page; keeps support, sales and leadership informed; protects the responders from interruptions | Make technical decisions |
| **Scribe** | Records the timeline with timestamps: observations, decisions, actions, results | Debug |
| **Subject-matter experts (SMEs)** | Database, network, security, payment provider experts, called in as needed | Take over coordination |
| **Customer/Executive liaison** (big incidents) | One person who talks to a key customer or execs, so they don't join the call | — |

### Where it comes from

The **Incident Command System (ICS)** was developed in the United States in the 1970s after large California wildfires showed that agencies couldn't coordinate. Its key ideas, adopted by tech companies like Google and PagerDuty, are:

- **One person in command** at any moment.
- **Clear roles** with a limited number of direct reports per leader.
- **Common language** (no team-specific jargon on the call).
- **Explicit handover** of command.

---

## Why it exists

In an outage without roles you typically see:

- Five senior engineers all debugging the same graph.
- Nobody updating the status page for 45 minutes.
- Two people restarting different things at the same time.
- An executive joining the call and asking "what's going on?" every five minutes, pulling the best engineer away from the fix.

Roles fix this by making sure **every important job has exactly one owner**.

---

## Analogy

**A film set.** The director (IC) decides what happens next and keeps everyone working toward one scene. The camera crew (Ops) does the technical work. The publicist (Comms) talks to the press. The script supervisor (Scribe) notes every take. If the director starts operating the camera, the scene falls apart.

---

## How it works

### Scaling the roles

| Incident size | Who does what |
|---|---|
| **SEV3, one service** | The on-call engineer is IC + Ops + Scribe (notes in the channel) |
| **SEV2** | On-call = Ops lead; a second person = IC + Comms; the chat channel is the timeline |
| **SEV1** | Separate IC, Ops lead, Comms lead, Scribe; SMEs join on request |
| **Huge SEV1 (many teams)** | IC + deputy IC; one Ops lead **per workstream** (e.g. "database recovery", "traffic", "customer comms"), each reporting to the IC |

### The IC's first 10 minutes (script)

1. "I'm the Incident Commander for this incident."
2. "What is the user impact right now?" → set severity.
3. "Who is Ops lead? Who is Comms? Who is Scribe?"
4. "What changed in the last few hours?" (deploys, configs, infrastructure, traffic, dependencies).
5. "What's the fastest safe thing we can do to reduce impact?" → decide.
6. "Next update at 14:30." (and keep it).

### Useful IC phrases

| Phrase | Why |
|---|---|
| "Let's do one change at a time. Ops, announce before you act." | So you know what helped |
| "What would make us wrong?" | Avoids tunnel vision on the first theory |
| "Who owns that? By when?" | Every action has a name and a time |
| "Can we take that offline?" | Moves deep discussions out of the main call |
| "Let's time-box this: if the rollback hasn't helped by 14:20, we fail over." | Prevents endless waiting |
| "Is anyone working on something I don't know about?" | Finds hidden parallel changes |

### Handing over command

ICs get tired. A long SEV1 needs fresh ICs every few hours. Do it **explicitly**:

```
Old IC: "Handover summary: impact is X, we've tried A and B, C is in progress,
         next decision point is 16:00. Comms has the status page at 15:30."
New IC: "I'm taking over as IC."
Old IC: "Confirmed, you have command."  (and announce it in the channel)
```

---

## Failure modes

| Failure | Symptom | Fix |
|---|---|---|
| **IC debugging** | Nobody coordinating; the IC goes quiet | IC hands off the keyboard; names an Ops lead |
| **No IC at all** | Chaos, duplicate work, no decisions | Declare an IC at the start of every SEV1/SEV2 |
| **Too many people talking** | 40 people on the call, constant interruptions | IC limits active speakers; others listen; side channels for SMEs |
| **Silent parallel work** | Someone quietly restarts the database "to help" | "Announce before you act" rule |
| **Executive takeover** | A VP starts giving technical orders | Comms lead / liaison briefs execs; IC keeps command |
| **No handover** | Exhausted IC after 10 hours makes a bad call | Rotate ICs, explicit handover |

---

## Where you'll use it

- **Big outages in Part 3:** Facebook 2021 (topic 59), Roblox 2021 (topic 70) and AWS 2025 (topic 62) were multi-team efforts that needed this structure.
- **Drills:** topic 90's capstone game day is run with these roles.
- **System Design:** [SD 30](../../System-Design/notes/30-incident-management.md) has the same roles in design-review form.

---

## Quick recap

- One person commands (IC); the IC coordinates and decides, but doesn't type.
- Ops lead fixes; Comms lead informs; Scribe records; SMEs help on request.
- Roles scale: one person can hold several in a small incident.
- One change at a time, announced. Time-box decisions.
- Hand over command explicitly.

### Interview questions

1. **Q: Why shouldn't the Incident Commander debug?**
   A: Because coordination is a full-time job during a big incident. If the IC is reading logs, nobody is tracking what everyone is doing, keeping the update cadence, spotting conflicting changes or making the call to try the next mitigation. The IC's value is the big picture.

2. **Q: You're the on-call and it's a SEV1. You're alone for the first 10 minutes. What do you do?**
   A: I act as IC and Ops at once, but I immediately page for help (a second engineer and an IC from the rota). While waiting, I post the impact in the incident channel, check what changed, and start the safest mitigation. When help arrives, I hand off either IC or Ops so the roles split.

3. **Q: An executive joins the incident call and starts asking for updates every few minutes. How do you handle it?**
   A: Politely redirect: the Comms lead (or a liaison) gives the executive a direct summary and the next update time, and the executive gets updates in a separate channel. That protects the responders while keeping leadership informed.

4. **Q: How do you hand over incident command?**
   A: Explicitly: the outgoing IC summarises impact, actions tried, actions in progress and the next decision points; the new IC says "I have command"; the change is announced in the channel and recorded by the scribe.
