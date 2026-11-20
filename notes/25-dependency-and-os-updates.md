# 25 · Dependency and OS Updates

> **Level:** ★★ intermediate
>
> **You will learn:** how changes you **didn't make** still break production: **unpinned dependencies** (`latest` tags, `^1.2` version ranges), **base image drift**, **unattended OS upgrades** on servers and Kubernetes nodes, **auto-updating agents** (security, monitoring), **managed service upgrades**, and **supply-chain** surprises (a package removed or compromised); how to find "the change nobody deployed", and how to make every update a **controlled, staged, reversible** change.
>
> **You need:** topics 01–24. Helpful: [Linux 031 · Packages](../../linux-mastery-rhel/notes/031-packages-rpm-dnf-and-apt.md), [Docker 19 · Base images](../../Docker/notes/19-choosing-base-images.md), [Docker 43 · Image supply chain](../../Docker/notes/43-image-supply-chain.md), [K8s 76 · Versions, skew and upgrades](../../CKA-Certification-Course-2025/notes/76-versions-skew-and-upgrades.md), [Terraform 62 · Upgrades](../../Terraform/notes/62-upgrades.md).
>
> **Part 2 · Recurring failures · C. Changes** · Previous: [24 · Bad config pushes](24-bad-config-pushes.md) · Next: [26 · DNS failures](26-dns-failures.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** "Nothing changed" is rarely true. Your software sits on top of **hundreds of things that update themselves**: OS packages, libraries pulled at build time, base images, agents, cloud-managed services. Any of them can change behaviour without a deploy from you.

```
Your code (pinned, reviewed)        ← you control this
  Libraries  (^1.4 = "any 1.x")     ← changes at next build
  Base image (python:3.12-slim)     ← changes when the tag is rebuilt
  OS packages (unattended-upgrades) ← changes overnight on every server
  Agents (security/monitoring)      ← vendor pushes updates
  Managed services (DB, K8s, LB)    ← provider upgrades on its schedule
```

Real examples: **Datadog, March 2023** (topic 63) and **Heroku, June 2025** (topic 64): automatic OS updates of systemd restarted networking and **removed network routes** on many hosts at once. **CrowdStrike, July 2024** (topic 52): a vendor-pushed content update to a security agent crashed Windows machines worldwide.

**Picture it.** You bake the same cake every week with the same recipe, but the supermarket changed the flour without telling you. The cake fails, and you swear you changed nothing.

**The one thing to know.** **Pin versions, and turn every update into a deploy**: built once, tested, rolled out in stages, with a rollback. Automatic updates applied to all production machines at the same time are a recipe for a fleet-wide outage.

**Try it now:** `grep -r "FROM" Dockerfile*` and `cat requirements.txt package.json` in one project. How many versions are unpinned?

**You've got it if you can:** find an unexpected package change on a host and describe a safe OS-patching process.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Symptoms

| Where | What you see |
|---|---|
| Timing | Failure starts at a **scheduled** time (e.g. 06:00 daily apt timer) or right after a rebuild, with **no app deploy** |
| Scope | Many hosts in the same **image/node pool/OS version** fail together |
| Builds | "It built yesterday, today the same commit fails/behaves differently" |
| Logs | New warnings/deprecations; library errors; `systemd` restarts in the journal; network routes missing |
| Kubernetes | Nodes NotReady in one node group (topic 21) |

---

## Analogy

The **supermarket flour** (Core).

---

## How it happens

| Type | Example | Why it bites |
|---|---|---|
| **Unpinned libraries** | `requests>=2`, `"lodash": "^4.17.0"`, no lockfile | A new minor version changes behaviour; builds aren't reproducible |
| **Mutable image tags** | `FROM node:20`, `image: nginx:latest` | The tag points to a new image after a rebuild upstream |
| **Unattended OS upgrades** | Ubuntu `unattended-upgrades`, `dnf-automatic` | Same update hits every host in the same window; some updates restart services (systemd, network, container runtime) |
| **Agent auto-updates** | Security/EDR agents, monitoring agents, log shippers | Run with high privileges, often in the kernel; vendor rolls out to all customers |
| **Managed upgrades** | Cloud DB minor version, managed Kubernetes auto-upgrade, CDN/WAF rule updates | Provider's schedule; behaviour change (defaults, deprecations) |
| **Removed/compromised packages** | npm `left-pad` unpublished (2016) broke many builds; the `xz-utils` backdoor (CVE-2024-3094, 2024) was found before wide distribution | You depend on other people's infrastructure and trust |
| **Provider/plugin updates** | Terraform provider minor version changing a default (Terraform 66 scenario 6) | Plan suddenly shows changes |

---

## How to find it

**First:** confirm there was no app deploy, and check for **scheduled** activity at the start time.

```bash
systemctl list-timers --all | grep -Ei "apt|dnf|unattended"
journalctl --since "2026-11-20 05:50" --until "2026-11-20 06:20" | grep -Ei "upgrade|install|restart|Started|Stopping"
```

**Second:** what packages changed on the host?

```bash
# Debian/Ubuntu
grep -E " (install|upgrade) " /var/log/dpkg.log | tail -20
zless /var/log/apt/history.log*      # with "Start-Date" and the package list
# RHEL
sudo dnf history list | head
sudo dnf history info <id>
rpm -qa --last | head -20
```
```
2026-11-20 06:04:12 upgrade systemd:amd64 255.4-1ubuntu8.4 255.4-1ubuntu8.5
2026-11-20 06:04:15 upgrade systemd-networkd ...
```
- systemd upgraded at 06:04; failures started 06:05 → strong suspect.

**Third:** compare a broken host with a healthy one (versions, routes, kernel).

```bash
uname -r; ip route; ip rule
dpkg -l | md5sum      # quick "are package sets identical?"
```

**For builds/images:**
```bash
docker image inspect myapp:v1.9 --format '{{.Created}} {{index .RepoDigests 0}}'
docker buildx imagetools inspect python:3.12-slim     # which digest does the tag point to today?
git diff HEAD~1 -- package-lock.json poetry.lock go.sum
```

---

## How to fix it

### Immediate

- **Stop the rollout of the update** (disable the timer/agent update policy fleet-wide) so healthy hosts don't get it.
- **Restore** affected hosts: revert the package (`dnf history undo <id>`, `apt install pkg=<old-version>`), restart the broken component, re-add routes — or **replace** hosts from the last known good image (faster and cleaner for cattle-style fleets).
- **Builds:** pin back to the last working version/digest and rebuild.
- **Vendor agent:** follow the vendor's remediation; isolate affected machines; open a priority case.

### Permanent

- **Pin everything:** lockfiles committed, base images by digest, provider versions pinned, `latest` banned in production.
- **Automated update PRs** (Renovate/Dependabot) so updates arrive as reviewed, tested changes.
- **Immutable infrastructure:** patch by building a **new image** (Packer, EKS AMI updates) and rolling it out **one node group/region at a time**, instead of patching in place.
- If you must patch in place: **stagger** (different windows per group), canary hosts first, `serial` with Ansible ([Ansible 45](../../Ansible/notes/45-strategies-forks-serial.md)), and hold packages that restart critical services (`apt-mark hold`, `dnf versionlock`) until tested.
- Vendor agents: use the vendor's staged-update / N-1 policies where available.
- Managed services: set maintenance windows, subscribe to release notes, test upgrades in staging first.

---

## How to verify the fix

- Affected hosts back to the known-good package set / image; service healthy.
- The update mechanism is paused or staged so it can't re-apply.
- Builds reproducible: rebuilding the same commit gives the same digest (or at least the same dependency versions).

---

## Prevention

- Inventory of every auto-update mechanism in production, each with an owner and a stage plan.
- Image and lockfile scanning in CI ([GHA 40 · Supply chain](../../GitHub-Actions-Basics-To-Production/notes/40-supply-chain-attestations.md), [Jenkins 59 · Trivy](../../Jenkins/notes/59-tool-primers-maven-sonar-trivy.md)).
- Canary node pool that receives updates first and is watched.
- Dashboards that annotate package/image changes, not only app deploys.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: unattended upgrades installed a systemd update on every node in the same window; restarting systemd-networkd removed routes that the CNI had added. Trigger: the package update. Root cause: production hosts patching themselves, simultaneously, with no staging.

2. **Q: How did you detect it? Why not earlier?**
   A: Many nodes NotReady at once. Not earlier because updates weren't treated as changes: no canary group, no annotation on dashboards.

3. **Q: What did you check first?**
   A: Whether there was an app deploy (no), then what changed on the hosts at the start time: package logs and timers.

4. **Q: How did you stop impact?**
   A: Disabled the update timers fleet-wide, replaced broken nodes from the last good image, and restored capacity.

5. **Q: Permanent fix?**
   A: Disabled in-place automatic upgrades; patches now ship as new node images, rolled out one node group at a time with health gates.

6. **Q: How did you confirm?**
   A: Nodes healthy; timers disabled everywhere (checked with Ansible ad-hoc); the next image rollout went smoothly.

7. **Q: What if reverting the package didn't fix it?**
   A: The update may have changed state that a revert doesn't restore (routes, config files, kernel needing reboot). Replace the host from a known-good image instead of repairing it.

8. **Q: Doesn't disabling auto-updates make us less secure?**
   A: Not if patches still ship quickly through the image pipeline. You still patch, but in a controlled, staged way. For urgent CVEs, the same pipeline runs faster, with a canary.

9. **Q: What would you do differently?**
   A: List every auto-update mechanism on day one, including vendor agents and managed services, and give each a staging plan.

10. **Q: What if the update came from a security vendor you can't control?**
    A: Use the vendor's update ring settings (N-1, staged groups) where available, keep a recovery runbook (e.g. booting into safe mode), and ask vendors about their staged rollout practices during procurement.
