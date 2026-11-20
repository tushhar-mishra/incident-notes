# 49 · Knight Capital, August 2012: $460 Million in 45 Minutes

> **Level:** ★ foundation
>
> **Pattern:** **Partial deploy** (one server of eight missed) + **reused feature flag** + **dead code** left in production + **no kill switch**. **Families:** change (bad deploy) · human error (manual deploy) · automation running at machine speed.
>
> **You will learn:** how a trading firm deployed new code to seven of eight servers, why the eighth server ran **old, dead code** when it saw a **repurposed flag**, how 212 customer orders became **millions** of orders and a **$460 million** loss in about 45 minutes, why 97 warning emails were ignored, why the team's emergency "rollback" made it **worse**, and what the SEC found.
>
> **You need:** Part 1, topics 23, 24.
>
> **Part 3 · Real outages · Bad deploys and config pushes** · Previous: [48 · Google Cloud × UniSuper 2024](48-google-cloud-unisuper-2024.md) · Next: [50 · Cloudflare regex 2019](50-cloudflare-regex-2019.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Knight Capital's order router (**SMARS**) contained old code called **Power Peg**, unused since 2003 but still present. New code for a new NYSE programme (**RLP**, Retail Liquidity Program) **reused the flag** that used to switch Power Peg on. The new code was deployed manually to 8 servers, but **one server didn't get it**. On launch day, orders with the flag reached that eighth server, which ran Power Peg — broken since a 2005 change — and it **never stopped sending orders**.

```
Flag X: old meaning = "run Power Peg"   new meaning = "run RLP"
Servers 1–7: new RLP code  ✅  flag X → RLP
Server 8:    OLD code       ❌  flag X → Power Peg (broken: doesn't know when an order is filled)
Aug 1, 9:30 ET: 212 parent orders → server 8 sends child orders endlessly
  → 4 million executions, 397 million shares, ~45 minutes → $460 million loss
Emergency fix: remove new code from servers 1–7  →  now ALL 8 run Power Peg  → worse
```

**Picture it.** A railway changes the meaning of a signal from "go to platform A" to "go to platform B". Seven signal boxes get the new rulebook; one doesn't. Trains arriving at that box are sent onto a track that leads nowhere — at full speed, one after another.

**The one thing to know.** **Never reuse a flag's meaning, delete dead code, automate deploys with verification, and have a kill switch** that stops the system when its output looks abnormal.

**You've got it if you can:** list the four separate failures that had to line up for this loss.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 1 August 2012, Knight Capital's SMARS order router began sending a flood of orders after the NYSE Retail Liquidity Program launched. New code had been deployed to seven of its eight SMARS servers; the eighth still contained defunct "Power Peg" code, which was activated by a flag that the new code had repurposed. In about 45 minutes, Knight executed over 397 million shares and lost more than $460 million.

## Impact

| Item | Detail (from the SEC order) |
|---|---|
| Orders | **212** incoming parent orders → millions of child orders → **4 million executions** in **154 stocks** |
| Volume | More than **397 million shares** in approximately **45 minutes** |
| Positions | ~**$3.5 billion net long** in 80 stocks and ~**$3.15 billion net short** in 74 stocks |
| Loss | **$460 million** realised loss; significant net capital problems |
| Market | For 75 stocks, Knight's executions were > 20% of volume and contributed to price moves > 5%; for 37 of those, moves > 10% |
| Regulatory | SEC charged violations of the Market Access Rule (Rule 15c3-5) and Regulation SHO; Knight agreed to pay a **$12 million** penalty (SEC press release) |

## Timeline (from the SEC order; times ET)

| When | Event |
|---|---|
| 2003 | Knight stops using Power Peg (code left in place, still callable) |
| 2005 | The "cumulative quantity" function (which stops sending child orders once a parent order is filled) is moved to an earlier point in the code; **Power Peg is not retested** |
| From 27 July 2012 | New RLP code deployed to SMARS servers in stages; a technician **does not copy the new code to one of the eight servers**; no second technician review; no written procedure requiring one |
| 1 Aug, from ~8:01 | Internal system sends **97 automated emails** ("BNET rejects") referencing SMARS and an error "Power Peg disabled" before the 9:30 open. Not designed as alerts; generally not reviewed |
| 1 Aug, 9:30 | Market opens. RLP-eligible orders with the repurposed flag reach server 8 → Power Peg runs, sending child orders without counting fills |
| ~45 minutes | Millions of executions accumulate. Staff try to find the cause while the system keeps trading |
| During | In one attempt, Knight **uninstalls the new RLP code from the seven good servers** → more parent orders activate Power Peg on those servers too, worsening the problem |

## How it happened

```
                          parent order (flag = "yes")
                                     │
          ┌──────────────────────────┼──────────────────────────┐
          ▼                          ▼                          ▼
   SMARS server 1..7           SMARS server 8 (missed by deploy)
   new RLP code: route          old Power Peg code:
   order correctly ✅            send child order → send child order → ...
                                 cumulative-quantity check was moved in 2005,
                                 so it never learns the parent is filled ❌
                                     │
                                     ▼
                     millions of child orders to exchanges
```

Four failures lined up:
1. **Dead code** left callable in production for nine years, broken by a 2005 change and never retested.
2. **A reused flag**: the new feature took over the old feature's switch.
3. **A manual, unverified deploy**: one of eight servers missed; no second reviewer; no automated check that all servers ran the same version.
4. **No output controls or kill switch**: nothing compared orders leaving SMARS with orders entering it, and nothing halted SMARS on its own abnormal activity.

## Why it spread

- **Machine speed:** automated trading turned an error into thousands of orders per second.
- **No aggregate limits:** the account collecting the unexpected positions (the "33 Account") had a $2 million gross limit but **wasn't linked to automated controls**; firm-wide capital thresholds didn't stop order entry.
- **Monitoring was manual:** the main risk tool (PMON) was post-execution, relied on humans, had no automated alerts, didn't show limits, and lagged under the high volume.
- **The "rollback" activated the bug everywhere**: removing the new code put the old code back on the seven good servers, where the repurposed flag again triggered Power Peg.
- **No incident procedures**: the SEC found Knight had no supervisory procedures to guide staff during significant incidents, and kept trading while investigating.

## How they found it

- **Before the open:** 97 emails referencing "Power Peg disabled" were the warning, but they weren't treated as alerts.
- **At the open:** senior staff saw a large volume of positions accumulating in the 33 Account in PMON, but couldn't quickly tell the source because that account held positions from many sources.
- The order does not give a minute-by-minute account of the diagnosis (**not publicly disclosed** in that detail).

## What they did to fix it

- Technology staff worked on the live system while it kept trading.
- One attempt — uninstalling the new code from the seven servers — **made it worse**.
- The SEC order does not detail the exact final step that stopped the orders (**not publicly disclosed** in the order).

## Permanent fix

The SEC order required Knight to retain an **independent consultant** to review its controls and procedures. The order's findings describe what was missing, which is the permanent-fix checklist:
- Controls immediately before orders reach the market (compare output with input; halt on aberrant activity).
- Firm-wide capital thresholds linked to automated controls that block order entry.
- Written code deployment procedures with a second reviewer, and protocols for unused code on production servers.
- Incident response procedures.
- Real reviews of risk (not just an inventory of existing controls).

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Manual deploys to a list of servers | Automated deploys (Ansible/ArgoCD/Jenkins) + a post-deploy check that **every** instance reports the same version ([Ansible 49 · Rolling deploy](../../Ansible/notes/49-project-rolling-deploy.md)) |
| Mixed versions in a Kubernetes Deployment | `kubectl get pods -o jsonpath` for image digests; alert on more than one version after rollout ends ([K8s 25](../../CKA-Certification-Course-2025/notes/25-deployments.md)) |
| Reusing a feature-flag name for new behaviour | New flag for new behaviour; remove old flags; flag lifecycle management ([SD 57](../../System-Design/notes/57-case-feature-flags-config.md)) |
| Dead code paths still callable | Delete them; coverage reports; "remove before reuse" rule |
| Warnings sent by email | Route real error signals to monitoring and alerts (topic 04) |
| Automation with no output limits | Rate limits, circuit breakers and kill switches on anything that acts on the outside world (payments, emails, orders) |
| "Rollback" that isn't really a rollback | Know what the previous version **does** with current flags/data; test rollback paths |

## Follow-up questions

1. **Q: What was the real root cause, not just the trigger?**
   A: The trigger was RLP orders arriving. The root causes were dead code left callable, a repurposed flag, a manual deploy with no verification that missed one server, and no automated output controls or kill switch.

2. **Q: How was it detected? Why wasn't it caught earlier?**
   A: 97 emails before the open pointed at the problem but weren't designed or treated as alerts. After the open, staff saw abnormal positions but the risk tool was manual and slow.

3. **Q: What should have been checked first?**
   A: What changed: the RLP deploy that week. Comparing the eight servers' versions would have shown the odd one out.

4. **Q: How should they have stopped the impact quickly?**
   A: Stop the system's output first (halt SMARS / disconnect from the market) — a kill switch — and then investigate. Investigating while trading continued is what made it so expensive.

5. **Q: What changed permanently?**
   A: The SEC required an independent review of controls; the findings call for pre-market controls, automated capital limits, deployment procedures and incident procedures.

6. **Q: How would you confirm a deploy like this is correct?**
   A: Automatic verification that every instance runs the new version (checksums/digests), plus a canary that compares behaviour before routing all traffic.

7. **Q: What if the fix (rollback) itself fails or makes it worse, as here?**
   A: Have a stop button that doesn't depend on understanding the bug: halt the system or cut its connection. Rollback is only safe if you know how the old version behaves with the new flags and data.

8. **Q: How would you prevent this class of incident?**
   A: Automated, verified deploys; delete dead code; never reuse flags; limits and kill switches on automated actions; incident procedures that say "stop first".

9. **Q: What would you do differently?**
   A: Treat any unexpected error message on launch day as a reason to delay the launch.

10. **Q: What if this happened in a payments system at 10× volume?**
    A: The same pattern (an automated system acting repeatedly on the outside world) would cause losses even faster. Idempotency keys, per-minute volume limits and automatic halts are mandatory for such systems.

## Source

- U.S. Securities and Exchange Commission, Administrative Proceeding, In the Matter of Knight Capital Americas LLC, Release No. 70694 (16 October 2013): <https://www.sec.gov/litigation/admin/2013/34-70694.pdf>
- SEC press release 2013-222, "SEC Charges Knight Capital With Violations of Market Access Rule": <https://www.sec.gov/news/press-release/2013-222>
