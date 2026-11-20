# 56 · Cloudflare, 5 December 2025: A Killswitch Meets an Old Bug

> **Level:** ★★ intermediate
>
> **Pattern:** A **safe-looking** change (turn off an internal test tool) through the **global configuration system** (no gradual rollout) → first-ever use of a **killswitch on an "execute" rule** → a **years-old nil-value bug** in Lua → HTTP 500 for ~28% of traffic for **25 minutes**. **Families:** change (config) · latent bug · old vs new code paths.
>
> **You will learn:** how a change made while protecting customers from a React vulnerability (CVE-2025-55182) caused a short global outage only 17 days after topic 55, why only customers on the **older FL1 proxy** with the **Managed Ruleset** were affected, and why Cloudflare **locked down all network changes** afterwards.
>
> **You need:** Part 1, topics 24, 55.
>
> **Part 3 · Real outages · Bad deploys and config pushes** · Previous: [55 · Cloudflare November 2025](55-cloudflare-nov-2025.md) · Next: [57 · Certificate expiry outages](57-certificate-expiry-outages.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Cloudflare was increasing its WAF's request **body buffer** (128 KB → 1 MB) to protect customers using React Server Components from a newly disclosed vulnerability. That change was going out **gradually**. An internal WAF **testing tool** didn't support the bigger buffer and wasn't needed, so engineers turned it off with a **second** change — via the **global configuration system**, which propagates to the whole fleet **within seconds**. Turning it off applied a **killswitch** to a rule with an **"execute"** action, something never done before, which hit a Lua bug that had existed **for years**.

```
Change 1: bigger WAF buffer          → gradual rollout ✓
Change 2: turn off WAF test tool     → GLOBAL config system (seconds, no gradual rollout)
   → killswitch skips an "execute" rule → later code expects rule_result.execute to exist
   → Lua: attempt to index field 'execute' (a nil value) → HTTP 500
   → only FL1 (old Lua proxy) + Managed Ruleset customers → ~28% of HTTP traffic
08:47 start → 08:50 declared (automated alerts) → 09:11 revert → 09:12 resolved (~25 min)
```

**Picture it.** You switch off a spare machine in a factory using the main power panel instead of the machine's own switch. The panel's wiring has an old fault that only shows up when that particular switch is used. Half the factory stops.

**The one thing to know.** The **riskiest** changes are often the "harmless" ones, pushed through the **fastest** path. Every path to production needs staged rollout — including killswitches and "just turning something off".

**You've got it if you can:** explain why the new Rust proxy (FL2) didn't fail and why that matters for migrations.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 5 December 2025, while rolling out a larger WAF body buffer to protect customers from a React Server Components vulnerability, Cloudflare used its global configuration system to disable an internal WAF testing tool. This applied a killswitch to a rule with an "execute" action for the first time, triggering a long-standing nil-value bug in the older FL1 proxy's Lua code. Customers on FL1 with the Cloudflare Managed Ruleset received HTTP 500 errors — about 28% of Cloudflare's HTTP traffic — for about 25 minutes until the change was reverted.

## Impact

| Item | Detail (from Cloudflare's postmortem) |
|---|---|
| Duration | ~**25 minutes** (08:47–09:12 UTC) |
| Affected | Customers served by the **older FL1 proxy** that had the **Cloudflare Managed Ruleset** deployed |
| Share | About **28% of all HTTP traffic** served by Cloudflare |
| Symptom | **HTTP 500** for all requests (except a few test endpoints like `/cdn-cgi/trace`) |
| Not affected | Customers without that configuration; China network traffic; the Rust FL2 proxy did not hit the error |
| Number of customers / cost | **Not publicly disclosed** |

## Timeline (UTC, from Cloudflare's postmortem)

| Time | Event |
|---|---|
| (before) | Change 1 (WAF buffer 128 KB → 1 MB, for CVE-2025-55182) rolling out through the gradual deployment system |
| 08:47 | Change 2 (disable internal WAF testing tool) deployed via the global configuration system; incident starts |
| 08:48 | Change fully propagated; full impact |
| 08:50 | Incident declared from **automated alerts** |
| 09:11 | Configuration change reverted; propagation starts |
| 09:12 | Revert fully propagated; all traffic restored |

## How it happened

```lua
-- simplified from the postmortem (FL1 proxy, Lua)
if rule_result.action == "execute" then
  rule_result.execute.results = ruleset_results[tonumber(rule_result.execute.results_index)]
end
```

```
Top-level ruleset ──"execute"──► ruleset of internal test rules
Killswitch (via global config) applied to that "execute" rule
   → evaluation of the execute action correctly SKIPPED
   → but later result processing still treats it as "execute"
   → rule_result.execute is nil → "attempt to index field 'execute' (a nil value)"
   → request processing fails → HTTP 500
```

Cloudflare: "we have never before applied a killswitch to a rule with an action of 'execute'"; the error "had existed undetected for many years". In the newer FL2 proxy, written in Rust, "the error did not occur".

## Why it spread

- **Global config system** = no gradual rollout; full impact within a minute. Cloudflare noted this system was already "under review following the outage we experienced on November 18, 2025", and that planned protections "would have helped prevent the impact" but weren't finished deploying.
- **Old code path:** the bug lived in the older FL1 proxy, still serving a large share of traffic during migration to FL2.
- **Hard-fail logic:** an error in rule processing failed the whole request instead of skipping the rule.

## How they found it

**Automated alerts** led to an incident being declared at 08:50, three minutes after the start. Cloudflare says the issue "was identified shortly after the change was applied" but doesn't give a time for identification (**not publicly disclosed**).

## What they did to fix it

1. **Reverted** the configuration change (09:11), fully propagated by 09:12.
2. **Locked down all changes to the network** while building better mitigation and rollback systems.

## Permanent fix (as listed by Cloudflare)

| Remediation | Pattern it addresses |
|---|---|
| **Enhanced Rollouts & Versioning**: data for rapid threat response and general configuration gets the same safety features as code (health validation, quick rollback) | Global config path without stages (topic 24) |
| **Streamlined break glass** capabilities for internal services and customer control-plane access | Access during incidents |
| **"Fail-Open" error handling**: replace hard-fail logic across critical data-plane components; on bad config, log and default to known-good or pass traffic without scoring; some services may let customers choose fail-open or fail-closed | Crash vs degrade |
| Network-wide **change lockdown** until better mitigation and rollback are in place | Stop the bleeding of change risk |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| A "quick" change via a fast path (kubectl edit, flag UI, Consul KV) while the "real" change goes through the pipeline | One pipeline for all production changes, with stages; fast path only for declared emergencies |
| Rarely used code paths (kill switches, "disable" options) | Test every switch position in CI and in game days |
| Nil/None handling in scripts (Lua in nginx/OpenResty, Python, Bash) | Defensive checks; linters; fail open for non-critical modules |
| Long migrations where old and new code both serve traffic | Prioritise migrating critical paths; keep the old path tested until it's gone |
| Making two changes at once during an emergency | One change at a time; announce each (topic 07) |

**Connects to:** [K8s 72 · Gateway API](../../CKA-Certification-Course-2025/notes/72-gateway-api-concepts.md) and [K8s 69 · Ingress](../../CKA-Certification-Course-2025/notes/69-ingress-concepts.md) — shared proxies serving many tenants are exactly where one bad rule hurts everyone.

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A years-old nil-value bug in FL1's rule-result processing, exposed for the first time by a killswitch on an "execute" rule, delivered through a configuration path with no gradual rollout. Disabling the test tool was the trigger.

2. **Q: How was it detected?**
   A: Automated alerts within three minutes; the incident was declared at 08:50.

3. **Q: What would you check first?**
   A: What changed at 08:47 — here a global config change made alongside an ongoing security rollout. With two simultaneous changes, separate them quickly.

4. **Q: How did they stop the impact?**
   A: Reverting the config change; recovery took one minute to propagate once reverted.

5. **Q: What changed permanently?**
   A: Staged rollouts and versioning for all configuration, better break-glass access, fail-open error handling, and a temporary lockdown of network changes.

6. **Q: How would you confirm the fix?**
   A: Test killswitches on every rule action type in CI; verify that a config error in the WAF fails open rather than returning 500.

7. **Q: What if the revert had failed?**
   A: The next step would be to disable the Managed Ruleset processing for affected traffic (fail open) or move traffic to FL2. That's why fail-open switches and alternate paths matter.

8. **Q: How would you prevent this class of incident?**
   A: No "special" fast path for config; every change staged and health-checked; regular testing of rarely used switches.

9. **Q: What would you do differently?**
   A: During an urgent security rollout, avoid unrelated changes; if a side change is needed, push it through the same gradual system.

10. **Q: What does this teach about rewriting systems (FL → FL2)?**
    A: Rewrites can remove whole classes of bugs (FL2 in Rust didn't have this error), but until migration is complete the old system's latent bugs remain live. Plan migrations with risk reduction on critical paths first.

## Source

- Cloudflare, "Cloudflare outage on December 5, 2025": <https://blog.cloudflare.com/5-december-2025-outage/>
