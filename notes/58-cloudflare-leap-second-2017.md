# 58 · Cloudflare, New Year 2017: When Time Went Backwards

> **Level:** ★★ intermediate
>
> **Pattern:** Code assumes **time only moves forward** + a **leap second** + measuring durations with the **wall clock** → a **negative number** → a **panic**. **Families:** certificates/time (topic 17).
>
> **You will learn:** how the leap second at the end of 31 December 2016 made some of Cloudflare's DNS servers compute a **negative round-trip time**, why `rand.Int63n()` **panicked** on it, why only **CNAME** lookups were affected (about **0.2% of DNS queries** at peak), how the fix was rolled out canary-first, and the general lesson: **use a monotonic clock for durations**.
>
> **You need:** Part 1, topic 17.
>
> **Part 3 · Real outages · Certificates and time** · Previous: [57 · Certificate expiry outages](57-certificate-expiry-outages.md) · Next: [59 · Facebook BGP 2021](59-facebook-bgp-2021.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Cloudflare's DNS server (**RRDNS**, written in Go) measured how fast each upstream resolver answered: `rtt := time.Now().Sub(start)`. At midnight UTC on 1 January 2017 a **leap second** was inserted, and on some machines the clock effectively went back. Now `time.Now()` could be **earlier** than `start`, so the duration was **negative**. After smoothing a few samples the value went negative, and the server-selection code passed it to `rand.Int63n()`, which **panics on negative input**. Some DNS lookups failed.

```
start = 23:59:59.900
leap second inserted → clock steps back
now   = 23:59:59.300  →  rtt = now - start = -0.6 s   ← "impossible" negative duration
smoothed average goes negative → rand.Int63n(negative) → panic → DNS resolution fails
```

**Picture it.** A stopwatch that uses the wall clock. If someone turns the wall clock back by a second while you're timing a runner, your stopwatch says they finished before they started — and the scoreboard software crashes on a negative time.

**The one thing to know.** **Measure durations with a monotonic clock**, never with wall-clock time, and never trust that a duration is positive.

**You've got it if you can:** explain why only CNAME lookups were affected and how a negative number caused a panic.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

At midnight UTC on 1 January 2017, the leap second caused Cloudflare's RRDNS software to compute negative round-trip times when measuring upstream DNS resolvers. A smoothed negative value reached `rand.Int63n()`, which panicked, so some DNS resolutions for Cloudflare-managed sites using CNAME records failed. At peak about 0.2% of Cloudflare's DNS queries were affected; the most affected machines were patched within 90 minutes and the fix was everywhere by 06:45 UTC.

## Impact

| Item | Detail (from Cloudflare's post) |
|---|---|
| Who | Only customers using **CNAME** DNS records with Cloudflare (Cloudflare resolves the origin's address on their behalf) |
| Scale | "approximately 0.2% of DNS queries" at peak; "less than 1% of all HTTP requests" encountered an error |
| Machines | "a small number of machines across Cloudflare's 102 data centers" |
| Duration | Impact 00:00–06:45 UTC; most affected machines patched within 90 minutes |
| Cost | **Not publicly disclosed** |

## Timeline (UTC, 1 January 2017)

| Time | Event |
|---|---|
| 00:00 | Impact starts (leap second at midnight) |
| 00:10 | Escalated to engineers |
| 00:34 | Issue confirmed |
| 00:55 | Mitigation on one canary node, confirmed |
| 01:03 | Mitigation in canary data center, confirmed |
| 01:23 | Fix in the most impacted data center |
| 01:45 → 01:48 | Fix deploying to major data centers, then everywhere |
| 02:50 | Fix on most affected data centers |
| 06:45 | Impact ends |

## How it happened

```
RRDNS upstream selection (for CNAME targets):
  for each upstream resolver:
     start := time.Now()
     ...query...
     rtt := time.Now().Sub(start)          // wall clock: NOT monotonic
     smoothedRTT = smooth(smoothedRTT, rtt)
  choose resolver with weights based on smoothedRTT
     rand.Int63n(weight)                    // panics if weight <= 0
```

At the time, the post says, Go's `time.Now()` "does not guarantee monotonicity" and Go didn't offer a monotonic time source. A single negative sample didn't break things, but "after a few measurements the smoothed value would eventually become negative." The panic was recovered by Go's `recover`, but the DNS resolution failed. Cloudflare named the deeper cause as the belief that "time cannot go backwards".

(Not from Cloudflare's post: Go 1.9, released later in 2017, added monotonic clock readings to `time.Now()` so `Sub` between two such values is safe against wall-clock steps — see the Go 1.9 release notes.)

## Why it spread

- Every RRDNS instance that experienced the time step was exposed at the same instant (midnight UTC).
- Only machines where the clock behaviour produced negative measurements were affected — a small number, so impact was limited.

## How they found it

The post says the problem "was quickly identified": escalated at 00:10, confirmed at 00:34. The detection method is **not publicly disclosed**.

## What they did to fix it

1. A small patch: check whether the time difference is negative and, if so, let RRDNS "forget about current upstream performance" and re-normalise, so negative numbers never reach the selection code.
2. **Canary first:** one node (00:55), one data center (01:03), the most impacted data center (01:23), then everywhere.
3. Restarted RRDNS servers after the fix to clear any lingering state.

## Permanent fix

Cloudflare committed to "inspecting all our code to ensure that there are no other leap second sensitive uses of time intervals."

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Timeouts/durations computed with `date +%s` in Bash scripts | Use `SECONDS` or monotonic sources; tolerate negative/zero durations |
| Application code using wall-clock time for latency, rate limits, TTLs | Monotonic clocks (`time.monotonic()` in Python, `System.nanoTime()` in Java, Go ≥ 1.9 `time.Now()` monotonic reading) |
| NTP stepping the clock on running hosts | chrony slewing, `makestep` only at boot; leap smearing (topic 17, [Linux 069](../../linux-mastery-rhel/notes/069-time-sync-chrony.md)) |
| Mixed smeared and non-smeared time sources | Use one consistent source per fleet |
| Random-weighted selection with computed weights | Validate inputs (`max(weight, 1)`) before calling APIs that panic |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Using wall-clock time to measure durations, plus code that assumed a duration can never be negative and passed it unchecked to a function that panics. The leap second was the trigger.

2. **Q: How was it detected, and why not earlier?**
   A: Detected within minutes after midnight; the detection method isn't public. It wasn't caught earlier because leap seconds are rare and the negative-time path wasn't tested.

3. **Q: What would you check first?**
   A: The time: errors starting exactly at 00:00 UTC on a leap-second date point straight at time handling.

4. **Q: How did they stop the impact?**
   A: A small patch that discards negative measurements, rolled out canary-first, then restarts.

5. **Q: What changed permanently?**
   A: An audit of all code for leap-second-sensitive use of time intervals.

6. **Q: How would you confirm the fix?**
   A: Unit tests that inject backwards time steps; chaos tests that step the clock on a canary host.

7. **Q: What if the patch had failed?**
   A: Fall back to a simpler selection method (e.g. round-robin) that doesn't depend on timing, until a correct fix is ready.

8. **Q: How would you prevent this class of incident?**
   A: Monotonic clocks for durations, defensive validation of computed values, and leap-smeared time sources.

9. **Q: What would you do differently?**
   A: Test time edge cases (leap seconds, DST changes, year boundaries, clock steps) before they happen.

10. **Q: Will leap seconds matter in future?**
    A: Fewer of them: in 2022 the international time authorities decided to stop adding leap seconds by 2035. But clock steps from NTP corrections, VM migrations and manual changes still happen, so the lesson stays.

## Source

- Cloudflare, "How and why the leap second affected Cloudflare DNS" (1 January 2017): <https://blog.cloudflare.com/how-and-why-the-leap-second-affected-cloudflare-dns/>
- Go 1.9 release notes, "Monotonic Time": <https://go.dev/doc/go1.9#monotonic-time>
