# 59 · Facebook, October 2021: The Network That Locked Itself Out

> **Level:** ★★ intermediate
>
> **Pattern:** A **maintenance command** + an **audit tool bug** → the whole **backbone** disconnected → **DNS servers withdraw their own BGP routes** → the internet can't find Facebook → **internal tools and remote access also down** → engineers must go **on site**. **Families:** DNS/network · human error/tooling · circular dependency (topic 38) · monitoring and access that share fate (topic 44).
>
> **You will learn:** how a command meant to **assess** backbone capacity took the backbone **down**, why that made Facebook's DNS disappear from the internet even though the DNS servers were running, why recovery needed **physical access** to secured data centers, why turning everything back on at once was **dangerous** (power, caches), and how "storm" drills helped.
>
> **You need:** Part 1, topics 26, 28, 38, 44.
>
> **Part 3 · Real outages · DNS and network** · Previous: [58 · Cloudflare leap second 2017](58-cloudflare-leap-second-2017.md) · Next: [60 · Cloudflare route leak 2026](60-cloudflare-route-leak-2026.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** During routine maintenance, an engineer ran a command to check the availability of global backbone capacity. It **unintentionally took down all backbone connections**, disconnecting Facebook's data centers from each other and from the internet. An audit tool should have stopped the command, but **a bug** let it through. Facebook's DNS servers are designed to **withdraw their BGP announcements** when they can't reach the data centers — so they did, and Facebook's names vanished from the internet.

```
maintenance command (meant to ASSESS capacity) ──► audit tool (bug: doesn't block it)
   ──► all backbone links down ──► data centers disconnected
   ──► DNS servers: "I can't reach the data centers → I'm unhealthy" → withdraw BGP routes
   ──► internet can't resolve facebook.com, instagram.com, whatsapp ...
   ──► internal tools (also need DNS/network) broken, remote + out-of-band access down
   ──► engineers sent on site; secure facilities and hardened routers slow them down
   ──► backbone restored → services brought back carefully (power, load)
```

**Picture it.** A building manager presses a button to "check the lifts" and accidentally locks every door in the building, including the doors to the control room. The building's sign then switches itself off, so visitors think the building has disappeared. Staff have to break in through security they designed to be hard to break.

**The one thing to know.** **Your recovery path must not depend on the thing that failed.** Out-of-band access, independent DNS, and incident tooling must survive a total network loss.

**You've got it if you can:** explain why the DNS servers were "operational but unreachable" and why recovery was slow.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

On 4 October 2021, a command issued during routine maintenance to assess global backbone capacity unintentionally took down all connections in Facebook's backbone network; a bug in the audit tool designed to catch such commands failed to stop it. With the backbone gone, Facebook's DNS servers withdrew their BGP advertisements, making Facebook's services unreachable from the internet, while internal tools and remote access also failed. Engineers restored the backbone on site and carefully brought services back.

## Impact

| Item | Detail (from Meta's engineering posts) |
|---|---|
| Services | Facebook and the company's other apps (Meta's update post covers "our services") were unavailable |
| Cause type | "there was no malicious activity behind this outage"; no evidence that user data was compromised |
| Duration, start/end times | **Not stated in Meta's posts** |
| Users affected / cost | **Not publicly disclosed** in Meta's posts |

## Timeline

Meta's posts give **no clock times**. The sequence they describe:

1. Routine maintenance: a command to assess backbone capacity.
2. The audit tool fails to stop it (bug); all backbone connections go down.
3. DNS locations declare themselves unhealthy and withdraw BGP advertisements → DNS unreachable from the internet.
4. Internal tools and normal and out-of-band network access are unavailable.
5. Engineers sent on site; secure access protocols activated.
6. Backbone brought back online; services restored gradually while managing power and load.

## How it happened

```
               Facebook backbone (connects all data centers)
   DC A ═══════════════ DC B ═══════════════ DC C
      ║                                        ║
   edge sites with authoritative DNS servers (well-known IPs, announced via BGP)
      │ rule: "if I can't talk to the data centers, withdraw my BGP announcements"
      ▼
   maintenance command → backbone links ALL down
      → DNS sites can't reach data centers → withdraw BGP → their IPs disappear from internet routing
      → resolvers worldwide: SERVFAIL / no answer for facebook.com
```

The DNS design choice is sensible in normal times: if a DNS site is cut off from the data centers, it should stop attracting traffic so users go to a healthy site. When **every** site was cut off at once, **every** site withdrew.

## Why it spread

- **Total backbone loss** = every data center affected at once.
- **DNS health logic** made a network failure into a global name-resolution failure.
- **Circular dependencies:** "the total loss of DNS broke many of the internal tools we'd normally use to investigate and resolve outages like this."
- **No remote way in:** "Our primary and out-of-band network access was down."
- **Security slowed recovery:** facilities and routers are designed to be hard to access and modify; activating secure access protocols to get people on site "took extra time". Meta's view: this trade-off (strong day-to-day security vs slower recovery from a rare event) is worth it.

## How they found it

Meta's posts do not describe the detection signals or what was checked first (**not publicly disclosed**). They describe confirming the issue once engineers had on-site access.

## What they did to fix it

1. Sent engineers to the data centers to debug and restart systems.
2. Restored backbone connectivity; "everything came back up with it".
3. **Careful restore:** turning everything on at once "could potentially cause a new round of crashes". Data centers had seen power usage dips "in the range of tens of megawatts", and suddenly reversing that could put "everything from electrical systems to caches at risk". Experience from **"storm" drills** (simulating taking a service, data center or region offline) helped them bring load back gradually; services came back "without any further systemwide failures".

## Permanent fix

Meta stated in general terms that a review was underway, that they'd look for ways to **simulate events like this** (they had never run a storm drill simulating the global backbone going offline), and would "strengthen our testing, drills, and overall resilience". **Specific fixes** (to the audit tool, command process, DNS withdrawal logic or out-of-band access) were **not publicly disclosed**.

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| A network change (security group, route table, NetworkPolicy) applied to everything at once | Stage it; dry-run; validate blast radius before apply (topic 28) |
| Health logic that withdraws capacity when it can't reach a dependency (LB health checks, readiness probes checking the DB) | Fail-open thresholds: if **everything** looks unhealthy, assume the checker is wrong (topic 29) |
| Bastion/VPN/SSO that depends on the production network | Independent break-glass access (cloud serial console, SSM via separate path, offline credentials) |
| Runbooks and chat on the same infrastructure | Offline copies; out-of-band communication (topic 06) |
| Restarting a whole cluster at once after an outage | Staged restarts; warm caches; admission control (topics 36–38) |
| Never practised "everything is down" | Game day for total loss of a region/network (topic 90) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: A maintenance command that could disconnect the entire backbone, combined with an audit tool bug that failed to block it. Contributing: DNS health logic that withdrew all sites when all were cut off, and recovery tools/access that depended on the same network.

2. **Q: How was it detected, and why not earlier?**
   A: Not described publicly. There was no chance to catch it "earlier": the damage was immediate, so prevention had to be in the command auditing.

3. **Q: What would you check first in a similar situation?**
   A: From outside: does DNS resolve, and are our prefixes visible in BGP (public looking glasses)? That separates "our servers are down" from "the internet can't find us".

4. **Q: How did they stop the impact?**
   A: Physically: on-site engineers restored the backbone. Then careful, gradual service restoration.

5. **Q: What changed permanently?**
   A: Only stated generally: reviews, more drills including backbone loss, stronger resilience. Specifics weren't published.

6. **Q: How would you confirm recovery is safe?**
   A: Bring load back in stages while watching power, cache hit rates and error rates, as Meta described.

7. **Q: What if on-site access had also failed?**
   A: Recovery would wait for physical access; that's why break-glass procedures for physical and network access need testing.

8. **Q: How would you prevent this class of incident?**
   A: Commands that can affect the whole network must be blocked by tested safeguards; changes staged; DNS and access designed to survive a backbone loss.

9. **Q: What would you do differently?**
   A: Practise the worst case (total network loss), including getting into the buildings, before it happens.

10. **Q: Why is "operational but unreachable" a useful phrase?**
    A: It reminds you to separate service health from reachability. The DNS servers worked but had told the internet not to route to them. Monitoring from inside said "fine"; users outside saw nothing.

## Source

- Meta Engineering, "More details about the October 4 outage" (5 October 2021): <https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/>
- Meta Engineering, "Update about the October 4th outage" (4 October 2021): <https://engineering.fb.com/2021/10/04/networking-traffic/outage/>
