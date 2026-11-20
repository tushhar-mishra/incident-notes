# 50 · Cloudflare, July 2019: One Regular Expression

> **Level:** ★★ intermediate
>
> **Pattern:** A **global config push** (a WAF rule) + **catastrophic regex backtracking** + a **removed safety limit**. **Families:** change (config push) · capacity (CPU exhaustion) · tooling that depends on the thing that's down.
>
> **You will learn:** how a small change to one Web Application Firewall rule used **100% CPU on every core** handling HTTP/HTTPS across Cloudflare's global network for **27 minutes**, what **catastrophic backtracking** is, why WAF rules skipped the staged rollout used for software, why engineers **couldn't log in** to fix it quickly, and the 11 contributing factors Cloudflare listed.
>
> **You need:** Part 1, topics 15, 24.
>
> **Part 3 · Real outages · Bad deploys and config pushes** · Previous: [49 · Knight Capital 2012](49-knight-capital-2012.md) · Next: [51 · Fastly 2021](51-fastly-2021.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Cloudflare's WAF rules are pushed **globally in seconds** so they can respond to new attacks. A new rule for detecting XSS (cross-site scripting) contained a **regular expression** that, on certain inputs, made the regex engine try a huge number of combinations (**backtracking**). Every server ran it on every request. CPUs hit ~100% worldwide; users got **502** errors for 27 minutes.

```
Rule merged → tests pass (no CPU test) → auto-deploy 13:42 UTC → Quicksilver pushes it worldwide in seconds
  → regex .*(?:.*=.*)) style pattern backtracks heavily → HTTP/HTTPS CPU ~100% on every machine
  → 502s everywhere → 14:07 global WAF kill → 14:09 traffic and CPU back to normal
```

**Picture it.** A new security guard instruction: "for each visitor, check every possible way to split their name into two parts, then check every way to split each part…". With short names it's fine. With long names, every guard in every building is stuck forever on the first visitor.

**The one thing to know.** **Rules and regexes are code.** They need performance tests, **staged rollout**, and an engine that can't explode (Cloudflare moved to regex engines with **run-time guarantees**).

**You've got it if you can:** explain backtracking in simple words and list three things that would each have prevented the outage.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 2 July 2019, a new rule in Cloudflare's WAF Managed Rules contained a regular expression that backtracked enormously. Deployed globally within seconds, it exhausted CPU on every core handling HTTP/HTTPS traffic across Cloudflare's network. Cloudflare's proxy, CDN and WAF returned 502 errors for 27 minutes until engineers globally disabled the WAF.

## Impact

| Item | Detail (from Cloudflare's postmortem) |
|---|---|
| Duration | Service "went down for 27 minutes" |
| What users saw | **502** error pages on Cloudflare-proxied domains |
| Scope | Global: core proxying, CDN and WAF functionality |
| Traffic | A leader reported verbally during the incident "we had lost 80% of our traffic" (not a measured figure) |
| Still working | Cloudflare's other protection mechanisms continued to operate |
| Customers / cost | **Not publicly disclosed** |

## Timeline (UTC, from Cloudflare's postmortem)

| Time | Event |
|---|---|
| 13:31 | Engineer merges an approved pull request with the rule change |
| 13:37 | CI (TeamCity) builds and runs tests: green (tests don't check CPU) |
| 13:42 | Automatic deployment begins |
| ~13:45 | First page: external synthetic WAF test failing; then many end-to-end test pages, global traffic drop alert, 502s, PoPs reporting CPU exhaustion. P0 declared. Initial speculation: a new kind of attack |
| 14:00 | WAF identified as the cause; attack ruled out (CPU data from a machine, `strace`) |
| 14:02 | Global WAF terminate proposed |
| 14:07 | Global WAF termination executed (delayed by access problems) |
| 14:09 | Traffic and CPU back to expected levels worldwide |
| 14:52 | Cause understood, fix in place; WAF re-enabled globally (after testing in one city with paying customers' traffic removed) |

## How it happened

```
The rule's regex contained the problematic part:   .*(?:.*=.*)
Simplified by Cloudflare to:                       .*.*=.*

Input "x=x"                    → 23 steps
Input "x=" + 20 x's            → 555 steps
Input of 20 x's with no "="    → 4,067 steps      (grows much faster than the input)
Lua WAF uses PCRE (backtracking engine) with NO protection against runaway expressions
```

**Backtracking in simple words:** `.*` means "match as much as possible". With `.*.*=.*`, the engine first lets the first `.*` grab everything, then finds no `=`, so it gives back one character and tries again, and again, trying every way to split the input between the two `.*` parts. The work grows much faster than the length of the input ("super-linearly"). On every request, on every server, this burned CPU.

Why it reached everyone at once:
- WAF rules are deployed via **Quicksilver**, a key-value store that pushes changes globally in seconds (Cloudflare: p99 of 2.29 s).
- Cloudflare's software releases go through staged environments (an employee-only PoP, a small subset of non-paying traffic, canary PoPs, then global). **The WAF SOP allowed rule changes to go global directly**, because WAF rules need to respond rapidly to threats.
- The change was meant to run in **"simulate" mode** (log but don't block). But in simulate mode the rule still **executes** — so it still burned CPU.

## Why it spread

Cloudflare listed eleven contributing factors (paraphrased):
1. A regex that could backtrack enormously was written.
2. A protection against excessive regex CPU use had been **removed by mistake** during a WAF refactoring weeks earlier.
3. The regex engine had no complexity guarantees.
4. The test suite couldn't detect excessive CPU.
5. The SOP allowed a non-emergency rule change to go globally without a staged rollout.
6. The rollback plan required running the complete WAF build twice, which took too long.
7. The first alert for the global traffic drop fired too slowly.
8. The status page wasn't updated quickly enough.
9. Accessing internal systems was difficult, and the bypass procedure wasn't well practised.
10. Some SREs had lost access because credentials time out for security when not used.
11. Customers couldn't reach the Cloudflare Dashboard or API, because they pass through the Cloudflare edge.

## How they found it

- **First signal:** external synthetic WAF test page, about three minutes after deploy start; then many more pages and a global traffic drop alert.
- **First suspicion:** an attack. Dismissed at 14:00 after the Performance Team examined CPU data and used `strace` on a machine.
- **Obstacle:** internal tools (control panel, Jira, build system) were behind the same edge; staff had to use a rarely used bypass.

## What they did to fix it

1. **Global WAF termination** (a mechanism to disable one component worldwide) at 14:07 → traffic and CPU normal by 14:09.
2. Fixed the rule and tested in a **single city** with paying customers' traffic removed.
3. Re-enabled the WAF globally at 14:52.
4. **Stopped all WAF release work** while remediation happened.

## Permanent fix (as listed by Cloudflare)

| Change | Pattern it addresses |
|---|---|
| Re-introduce the excessive-CPU protection that was removed (done) | Lost safety limit |
| Manually inspect all 3,868 WAF Managed Rules for backtracking (done) | Latent problems elsewhere |
| Performance profiling for all rules in the test suite | Tests that measure CPU |
| Switch to **re2 or Rust regex** engines, which have run-time guarantees | Make the failure impossible |
| Change the SOP to **staged rollouts** for rules, keeping emergency global deployment for active attacks | Blast radius |
| Emergency ability to take the Dashboard and API **off Cloudflare's edge** | Tools depending on the broken system |
| Automate Cloudflare Status page updates | Communication |
| Longer term: move from the Lua WAF to the new firewall engine | Architecture |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Regex in nginx/ingress rules, WAF (ModSecurity, AWS WAF), log parsers (Logstash grok), alert rules, input validation | Avoid nested `.*`; use RE2-style engines where possible (Go's `regexp` and Rust's `regex` are linear-time); fuzz/benchmark regexes in CI |
| Config pushed to all pods at once (ConfigMap + reload sidecar, feature flag) | Staged rollout of config (topic 24) |
| CI that only checks correctness | Add performance tests / time budgets for rule sets |
| Kill switch you can't reach during the incident | Break-glass access outside the main path; test it (topic 44) |
| Credentials that expire when unused | Break-glass accounts that are tested regularly |

**Connects to:** [Linux 017 · grep and regular expressions](../../linux-mastery-rhel/notes/017-grep-and-regular-expressions.md), topic 15 (CPU), topic 24 (config pushes).

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A regex with catastrophic backtracking, running in an engine with no protection (the CPU limit had been removed by mistake), deployed globally at once because WAF rules skipped staged rollout. The rule change was the trigger.

2. **Q: How was it detected? Why not earlier?**
   A: External synthetic tests ~3 minutes after deploy. Earlier detection needed a CPU/performance test in CI and a staged rollout that would show CPU rising in one location first.

3. **Q: What was checked first?**
   A: Whether it was an attack (the symptoms looked like one); CPU profiling and `strace` on a machine pointed to the WAF.

4. **Q: How did they stop impact?**
   A: A global kill switch for the WAF, which restored traffic in about two minutes.

5. **Q: What changed permanently?**
   A: CPU protection restored, all rules audited, performance tests, linear-time regex engines, staged rollouts for rules, independent access paths, automated status updates.

6. **Q: How did they confirm the fix?**
   A: Testing the fixed rule in a single city with paying traffic removed before re-enabling globally.

7. **Q: What if the global kill switch had failed?**
   A: The next option was a full rollback, which needed the WAF build twice and was slow (a listed contributing factor). That's why fast, practised kill switches matter.

8. **Q: How would you prevent this class of incident?**
   A: Treat rules/config as code with performance tests and staged rollout, and choose engines that can't have super-linear behaviour.

9. **Q: What would you do differently?**
   A: Practise the bypass and kill-switch procedures regularly, so access problems don't add minutes.

10. **Q: Why didn't "simulate mode" protect them?**
    A: Simulate mode only changes the **action** (log instead of block). The rule still **runs** on every request, so its CPU cost is the same. "Safe" modes must be judged by what still executes.

## Source

- Cloudflare, "Details of the Cloudflare outage on July 2, 2019": <https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/>
