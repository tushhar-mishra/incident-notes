# 75 · CircleCI, January 2023: Rotate Every Secret

> **Level:** ★★ intermediate
>
> **Pattern:** **Malware** on one engineer's laptop steals a valid **2FA-backed SSO session** → attacker accesses production → **exfiltrates customer secrets** (environment variables, tokens, keys) → encryption at rest doesn't help (keys taken from a **running process**) → every customer must **rotate every secret**. **Families:** security incident · secrets (topic 42) · CI/CD platform as a high-value target.
>
> **You will learn:** why your **CI/CD system** holds the keys to everything, how session-cookie theft defeats 2FA, why "encrypted at rest" didn't protect the data, how a **customer's alert** started the investigation, what CircleCI did (rotate OAuth tokens with GitHub and Atlassian, revoke API tokens, notify AWS customers), what customers had to do (**rotate at the source**, review logs), and why **OIDC** short-lived credentials are the long-term answer.
>
> **You need:** Part 1, topics 02, 42.
>
> **Part 3 · Real outages · Automation and security** · Previous: [74 · Cloudflare BYOIP 2026](74-cloudflare-byoip-2026.md) · Next: [76 · How to run drills](76-how-to-run-drills.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** CI/CD platforms store secrets for deploying to clouds, registries, package managers and more. On 16 December 2022, malware on a CircleCI engineer's laptop (not detected by antivirus) stole a **valid, 2FA-backed SSO session**. The attacker impersonated the engineer, who could generate production access tokens, and on 22 December **exfiltrated customer environment variables, tokens and keys**. The data was encrypted at rest, but the attacker **extracted encryption keys from a running process**. A **customer** reported suspicious GitHub OAuth activity on 29 December. On 4 January 2023, CircleCI told **all customers to rotate all secrets**.

```
Dec 16  malware on engineer laptop → steals SSO session cookie (2FA already completed)
Dec 19  attacker reconnaissance in production (impersonating the engineer)
Dec 22  exfiltration: customer env vars, tokens, keys (+ encryption keys from a running process)
Dec 29  a customer reports suspicious GitHub OAuth activity → investigation with GitHub
Dec 31  CircleCI starts rotating all GitHub OAuth tokens
Jan 4   disclosure: "Immediately rotate any and all secrets stored in CircleCI"
Jan 4–12 production access restricted; hosts rotated; API tokens revoked; Bitbucket tokens rotated; AWS notifications
```

**Picture it.** A building's master-key cabinet is in the security office. A thief copies a guard's badge (already authenticated) from a hacked phone, walks into the office looking like the guard, and photographs every key. The keys were in a locked box — but the box's own key was hanging next to it. Every tenant now has to change their locks.

**The one thing to know.** **Your CI/CD system is one of the most valuable targets you have.** Prefer **short-lived credentials (OIDC)** so there's little to steal, and know how to **rotate everything** quickly.

**You've got it if you can:** explain why "rotate at the source" matters and why OIDC reduces the damage of a CI breach.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## Summary

In December 2022, an attacker used malware on a CircleCI engineer's laptop to steal a valid, 2FA-backed SSO session, escalated access to a subset of production systems, and exfiltrated customer environment variables, tokens and keys, extracting encryption keys from a running process to potentially decrypt them. After a customer reported suspicious GitHub OAuth activity on 29 December, CircleCI investigated, rotated tokens with partners, and on 4 January 2023 asked all customers to rotate every secret stored in CircleCI.

## Impact

| Item | Detail (from CircleCI's report) |
|---|---|
| Data | "customer environment variables, tokens, and keys" from a subset of databases and stores |
| Encryption | Encrypted at rest, but the attacker "extracted encryption keys from a running process" |
| Confirmed downstream misuse | At publication, "fewer than 5 customers" had reported unauthorized access to third-party systems as a result |
| Customers' action | Rotate all secrets; review logs for suspicious activity from 16 December 2022 until their rotation was complete |
| Number of customers whose data was taken | **Not publicly disclosed** |

## Timeline (UTC, from CircleCI's report and security alert)

| When | Event |
|---|---|
| 16 Dec 2022 | Engineer's laptop compromised with malware (not detected by antivirus) |
| 19 Dec | Attacker reconnaissance in production |
| 22 Dec | Exfiltration — CircleCI's last record of unauthorized activity in production |
| 29 Dec | A customer alerts CircleCI to suspicious GitHub OAuth activity; investigation with GitHub begins |
| 30 Dec | That customer's GitHub OAuth token found compromised |
| 31 Dec 04:00 | CircleCI begins rotating all GitHub OAuth tokens on customers' behalf |
| 4 Jan 2023 | 16:35 the employee's account access shut down; 18:30 production access shut down for nearly all employees; 22:30 potentially exposed production hosts rotated |
| 5 Jan 02:30 | Disclosure emails and blog post (6:30 PM PST, 4 Jan) |
| 5 Jan 03:26 | All Project API Tokens revoked |
| 6 Jan | Personal API Tokens created before 5 Jan revoked; Bitbucket tokens rotated with Atlassian |
| 7 Jan | GitHub OAuth rotation completed; work with AWS to notify customers about potentially affected AWS tokens (complete 12 Jan) |
| 12–13 Jan | Full incident report published |

## How it happened

```
Engineer laptop ── malware (undetected) ── steals SSO session cookie (2FA already passed)
                                              │
Attacker (remote) ── reuses session ── impersonates engineer ── engineer can generate production access tokens
                                              │
                                Production: databases/stores with customer secrets (encrypted at rest)
                                              │  + encryption keys read from a running process
                                              ▼
                                    exfiltrated env vars, tokens, keys → potential use against customers' clouds/repos
```

CircleCI: "a security incident is a systems failure", and the employee's privileges to generate production access tokens were part of their regular duties — the issue was what the system allowed a stolen session to do.

## Why it spread

- **A single session** gave access to systems holding **many customers'** secrets.
- **Long-lived customer secrets** (cloud keys, API tokens) remain useful to attackers until rotated.
- **Encryption at rest** didn't help once the attacker could read keys from running processes.
- **Detection came from a customer**, after exfiltration.

## How they found it

"On December 29, 2022, we were alerted to suspicious GitHub OAuth activity by one of our customers." This started a deeper review with GitHub. The report says findings were based on authentication, network and monitoring tools plus logs from partners; how CircleCI's own monitoring first flagged the intrusion is **not publicly disclosed**.

## What they did to fix it

1. Rotated **all GitHub OAuth tokens** (with GitHub raising API rate limits to allow it).
2. Shut down the employee's access and restricted production access to very few employees; rotated production hosts.
3. **Revoked** all Project API Tokens and older Personal API Tokens.
4. Rotated **Bitbucket** tokens with Atlassian; notified customers about potentially affected **AWS** tokens with AWS.
5. Released tools: a script to find secrets in CircleCI, checksums for checkout keys, `updated_at` in the Contexts API, audit logs for all plans.
6. Asked all customers to **rotate all secrets** — "at the source (the system to which they provide access) and then store the new secrets on CircleCI. Simply removing them from CircleCI is not enough."

## Permanent fix (as listed by CircleCI)

| Change | Pattern it addresses |
|---|---|
| Detection and blocking via MDM and antivirus for the malware's specific behaviours | Endpoint security |
| Production access restricted to very few employees; **step-up authentication** for those who keep it | Least privilege |
| Monitoring and alerting for the behaviour patterns seen | Detection |
| Planned: periodic automatic OAuth token rotation; moving from OAuth to **GitHub Apps**; third-party review of tooling configurations; reducing session trust, more authentication factors, more regular access rotation; more ephemeral permissions | Long-lived credentials and sessions |
| Making it easier for customers to adopt **OIDC** and IP ranges | Short-lived customer credentials |

## In my stack

| Same risk in my tools | How I'd prevent it |
|---|---|
| Static AWS keys in Jenkins credentials / GitHub Actions secrets | **OIDC** federation to cloud roles; no long-lived keys ([GHA 36](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md)) |
| One CI system with secrets for every environment | Separate credentials per environment; GitHub **environments** with protection rules; least-privilege roles ([GHA 28](../../GitHub-Actions-Basics-To-Production/notes/28-environments.md), [Jenkins 23](../../Jenkins/notes/23-credentials-model.md)) |
| Engineers with standing admin access to production | Just-in-time elevation; step-up auth; short session lifetimes |
| No inventory of where secrets are used | Secrets manager with inventory and rotation; know how to rotate everything in hours (topic 42, [SD 54](../../System-Design/notes/54-case-secrets-management.md)) |
| Self-hosted CI (Jenkins) as a crown jewel | Harden and isolate it ([Jenkins 65 · Securing Jenkins](../../Jenkins/notes/65-securing-jenkins.md)) |

## Follow-up questions

1. **Q: What was the real root cause?**
   A: An attacker reused a stolen, already-authenticated session to act as an engineer who could generate production tokens, and production held many customers' long-lived secrets whose encryption keys were reachable from running processes. The malware was the entry point.

2. **Q: How was it detected, and why so late?**
   A: A customer reported suspicious GitHub OAuth activity a week after exfiltration. Antivirus missed the malware, and session reuse from a new location wasn't flagged.

3. **Q: As a customer, what would you check first?**
   A: Which secrets you stored in CircleCI (projects and contexts), then the logs of each target system (cloud, Git, registries) for unusual activity from 16 December onwards.

4. **Q: How was the impact limited?**
   A: Mass rotation and revocation of tokens with partners, restricted production access, and asking every customer to rotate everything.

5. **Q: Why "rotate at the source"?**
   A: Deleting a secret from CircleCI doesn't invalidate the stolen copy. Only changing it in the system it unlocks (e.g. creating a new AWS key and deactivating the old one) makes the stolen value useless.

6. **Q: What changed permanently?**
   A: Endpoint detection, restricted and step-up access, monitoring, and plans for token rotation, GitHub Apps, reduced session trust, and easier OIDC adoption.

7. **Q: How would you confirm your rotation is complete?**
   A: Inventory every secret, check each old credential fails, and watch target-system logs for use of old credentials.

8. **Q: What if you can't find every place a secret was used?**
   A: Rotate anyway and watch for failures; that's why a central secrets inventory and short-lived credentials matter.

9. **Q: How does OIDC reduce the damage of a CI breach?**
   A: The CI system holds no long-lived cloud keys; each job gets a short-lived token scoped to a role and conditions (repo, branch, environment). A stolen database dump contains nothing reusable later.

10. **Q: What does "a security incident is a systems failure" mean in practice?**
    A: Same as blameless postmortems (topic 11): the question isn't "who got phished" but why one compromised laptop could reach so much, and how to limit that by design.

## Source

- CircleCI, "CircleCI incident report for January 4, 2023 security incident": <https://circleci.com/blog/jan-4-2023-incident-report/>
- CircleCI, "CircleCI security alert: Rotate any secrets stored in CircleCI" (updated through 13 January 2023): <https://circleci.com/blog/january-4-2023-security-alert/>
