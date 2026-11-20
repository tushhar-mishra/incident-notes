# 17 · Clock Skew and NTP

> **Level:** ★★ intermediate
>
> **You will learn:** why servers' clocks **drift**, how **NTP** (Network Time Protocol) and **chrony** keep them right, what **clock skew** breaks (**TLS** certificates, **signed API requests**, **Kerberos**, **one-time passwords**, **JWT** tokens, **distributed databases**, log ordering), **wall clock vs monotonic clock**, **leap seconds**, how to check time on a host and in containers, and how to fix a skewed clock **safely** (step vs slew).
>
> **You need:** topics 01–16. Helpful: [Linux 069 · Time sync with chrony](../../linux-mastery-rhel/notes/069-time-sync-chrony.md), [SD 26 · Distributed systems realities](../../System-Design/notes/26-distributed-systems-realities.md).
>
> **Part 2 · Recurring failures · A. Linux host resources** · Previous: [16 · Memory leaks and OOM](16-memory-leaks-and-oom-killer.md) · Next: [18 · OOMKilled in Kubernetes](18-oomkilled-in-kubernetes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Every computer clock drifts a little. NTP/chrony constantly corrects it against reference servers. If time sync stops, the clock **skews** (differs from real time), and anything that checks timestamps starts failing in confusing ways.

```
Real time:   12:00:00
Server A:    12:00:00   ✅
Server B:    12:07:30   ❌ 7.5 min ahead
  → AWS API calls from B rejected ("Signature expired" / RequestTimeTooSkewed)
  → TLS: certificates "not yet valid" or "expired" if far enough off
  → Kerberos login fails (default tolerance 5 min)
  → TOTP 2FA codes rejected
  → logs from A and B appear in the wrong order
```

**Picture it.** A group of friends meeting "at 7". One watch is 15 minutes fast. That person arrives early, leaves early, and everyone's story of the evening doesn't line up.

**The one thing to know.** When errors are about **"expired"**, **"not yet valid"**, **"signature"** or **"token"**, and only **some** servers have them, check the **clock** on those servers.

**Try it now:** `timedatectl` and `chronyc tracking`.

**You've got it if you can:** check whether a host is synced, by how much it's off, and fix it without breaking running apps.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Cloud APIs | AWS: `Signature expired`, `RequestTimeTooSkewed`, `The difference between the request time and the current time is too large` |
| TLS | `x509: certificate has expired or is not yet valid` on one host only |
| Auth | Kerberos `Clock skew too great`; JWT `token used before issued` / `nbf` errors; TOTP codes rejected |
| Databases / distributed systems | Strange ordering, rejected writes, lease/lock problems in systems that rely on time |
| Logs | Events out of order across hosts; traces with negative durations |
| Monitoring | Metrics with timestamps in the future (dropped or misplaced); Prometheus "out of bounds" |
| Scheduling | Cron jobs run at the wrong time; caches expire too early/late |

---

## Analogy

The **friends with a fast watch** (Core). Leap seconds are like the whole city agreeing to stop the clocks for one second; code that assumed "time always moves forward by exactly one second each second" gets confused.

---

## How it happens

- **Time sync not running:** chronyd/ntpd stopped, not installed in an image, or disabled in a VM template.
- **Can't reach NTP servers:** firewall blocks UDP 123 outbound; DNS for the NTP pool fails; a private network with no time source.
- **Bad source:** an NTP server serving wrong time; a VM restored from snapshot or resumed after a pause with an old clock.
- **Virtualisation:** VM paused/migrated; hypervisor time sync conflicting with NTP.
- **Big correction applied as a step:** the clock jumps backwards or forwards, breaking code that measures durations with wall-clock time.
- **Leap seconds:** an extra second (23:59:60) inserted at the end of June or December. Bugs in 2012 (Linux kernel livelock causing high CPU in Java and MySQL) and 2017 (Cloudflare DNS, topic 58) came from this. The international time authorities decided in 2022 (CGPM, General Conference on Weights and Measures) to stop adding leap seconds by 2035. Many large providers already **smear** leap seconds (spread the extra second over hours).

### Wall clock vs monotonic clock

| Clock | What it is | Use it for |
|---|---|---|
| **Wall clock** (`CLOCK_REALTIME`) | Human date/time; can jump when NTP steps it | Timestamps shown to humans, certificates, expiry dates |
| **Monotonic clock** (`CLOCK_MONOTONIC`) | Always moves forward; not affected by NTP steps | Measuring durations, timeouts, rate limits |

Code that measures "how long did this take" with the wall clock can see **negative durations** after a backward step. That is exactly what hit Cloudflare in 2017 (topic 58).

### Step vs slew

- **Slew:** speed up or slow down the clock slightly until it's correct (chrony slews by default for small offsets). Safe; slow for big offsets.
- **Step:** jump straight to the right time. Fast; can break running software.

---

## How to find it

**First:** is this host synced?

```bash
timedatectl
```
```
               Local time: Thu 2026-11-20 14:10:03 UTC
           Universal time: Thu 2026-11-20 14:10:03 UTC
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: no
              NTP service: inactive
```
- `synchronized: no` and `NTP service: inactive` → time sync is **off**.

**Second:** how far off, and from what source?

```bash
chronyc tracking
```
```
Reference ID    : A9FEA97B (169.254.169.123)
Stratum         : 4
System time     : 452.318 seconds fast of NTP time
Last offset     : +0.000012 seconds
Leap status     : Normal
```
- `System time: 452 seconds fast` → **7.5 minutes ahead**. That breaks AWS signatures and Kerberos.
- `169.254.169.123` is the Amazon Time Sync Service (link-local address available inside EC2).
- `Leap status: Not synchronised` would mean chrony has no working source.

```bash
chronyc sources -v
```
```
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^? 169.254.169.123               0   6     0     -     +0ns[   +0ns] +/-    0ns
```
- `^?` and `Reach 0` → the source has **never answered** (blocked firewall, wrong address).

**Third:** compare hosts quickly.

```bash
for h in web1 web2 web3; do echo -n "$h "; ssh $h date -u +%s; done
date -u +%s
```
Or in Prometheus with node_exporter: `node_timex_offset_seconds` and `node_timex_sync_status`.

**Containers:** containers use the **host's** kernel clock (there's no separate clock to fix inside a normal container). Fix time on the **node**.

---

## How to fix it

### Immediate

```bash
sudo systemctl enable --now chronyd          # RHEL; 'chrony' service on Debian/Ubuntu
sudo chronyc makestep                         # step now (only if the apps can tolerate it)
chronyc tracking                              # check offset is ~0
```
- If the offset is **huge** and the host runs time-sensitive software (database, consensus), consider draining it from service first, then stepping, then returning it.
- Take the skewed host **out of the load balancer** while fixing, so users stop hitting errors.

### Permanent

- chrony enabled in every image (Packer/Ansible baseline) with reachable sources: cloud provider time service first (Amazon Time Sync, Google's metadata server NTP, Azure host time), then internal servers.
- Allow UDP 123 to time sources in security groups/firewalls.
- `makestep 1.0 3` in `chrony.conf` (step only during the first 3 updates after boot, when apps aren't running yet, then slew).
- Code: use monotonic clocks for durations; allow small skew tolerance in token validation (`leeway`).

---

## How to verify the fix

- `timedatectl` → `System clock synchronized: yes`.
- `chronyc tracking` → offset in milliseconds or less; `Leap status: Normal`.
- Errors (signatures, TLS, tokens) stop from that host.
- Fleet-wide: max `node_timex_offset_seconds` small.

---

## Prevention

- Alert: `node_timex_sync_status == 0` for 10 min, or `abs(node_timex_offset_seconds) > 0.05` (tune to your needs).
- Time sync is part of the base image and the production readiness checklist.
- Test VM restore/resume procedures for time correction.
- Use leap-smearing time sources consistently (don't mix smeared and non-smeared sources).

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: The trigger was signature errors from one host. The root cause was that chronyd wasn't running on hosts built from a new image (the package was missing), so their clocks drifted over weeks. Contributing: no time-sync alert, and no image test for it.

2. **Q: How did you detect it? Why not earlier?**
   A: Application errors ("Signature expired") from a subset of hosts. Drift is slow, so it took weeks to cross AWS's tolerance; with an offset alert it would have been caught at a few hundred milliseconds.

3. **Q: What did you check first?**
   A: Whether the errors came from specific hosts, then `timedatectl`/`chronyc tracking` on one of them, because "expired/signature" errors on only some hosts strongly point to time.

4. **Q: How did you stop the impact?**
   A: Took the affected hosts out of the load balancer, started chronyd and stepped the clock, then returned them.

5. **Q: How did you fix it permanently?**
   A: Added chrony to the base image with the cloud time source, an image test that checks sync, and an offset alert.

6. **Q: How did you confirm?**
   A: Offsets near zero on all hosts; no new signature errors; sync status 1 everywhere.

7. **Q: What if chrony starts but never syncs?**
   A: `chronyc sources` shows `Reach 0`: the source is unreachable. Check UDP 123 egress, the source address (link-local cloud time service vs public pool), DNS for pool names, and SELinux/firewalld rules.

8. **Q: Why might stepping the clock be dangerous?**
   A: Software that measures durations or timeouts with wall-clock time can see time jump backwards (negative durations) or forwards (everything times out). Databases and consensus systems may misbehave. Prefer slewing, or drain the host first.

9. **Q: What would you do differently?**
   A: Treat time as a monitored dependency like DNS: alert on offset and sync status from day one.

10. **Q: What if it happened on Kubernetes nodes?**
    A: Pods use the node's clock, so fix the node (DaemonSet/node image), not the pods. Skew on control-plane nodes can affect certificate validation and leader-election leases; fix those first.
