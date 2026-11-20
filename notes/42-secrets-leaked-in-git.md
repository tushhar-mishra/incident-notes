# 42 · Secrets Leaked in Git

> **Level:** ★★ intermediate
>
> **You will learn:** how secrets (cloud keys, database passwords, API tokens) end up in **Git**, CI logs, container images and Terraform state; why a leaked key in a **public** repo can be abused **within minutes**; the right **order** of response (**revoke/rotate first**, investigate, then clean history); how to check what an attacker did (**CloudTrail**); why deleting the commit is **not** enough; and prevention with **secret scanning**, **pre-commit hooks**, **OIDC** short-lived credentials and secret managers.
>
> **You need:** topics 01–41. Helpful: [GHA 17 · Secrets and variables](../../GitHub-Actions-Basics-To-Production/notes/17-secrets-and-variables.md), [GHA 36 · OIDC cloud authentication](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md), [Jenkins 24 · Using credentials safely](../../Jenkins/notes/24-using-credentials-safely.md), [Docker 44 · Handling secrets](../../Docker/notes/44-handling-secrets.md), [Terraform 51 · Secrets](../../Terraform/notes/51-secrets.md), [Ansible 40 · Vault](../../Ansible/notes/40-ansible-vault.md), [SD 37 · Config and secrets at scale](../../System-Design/notes/37-config-and-secrets-at-scale.md).
>
> **Part 2 · Recurring failures · G. Human error, security, cost** · Previous: [41 · Terraform destroying production](41-terraform-destroying-production.md) · Next: [43 · Runaway cloud costs](43-runaway-cloud-costs.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A secret pushed to Git is **compromised**, even if you delete it a minute later. Public repositories are scanned continuously by bots looking for keys. The only safe response is: **revoke and rotate the secret first**, then find out whether it was used, then clean up.

```
1. REVOKE / ROTATE      (minutes)   disable the key, issue a new one, update apps
2. INVESTIGATE          (hours)     was it used? by whom? what did they do? (CloudTrail, provider logs)
3. CONTAIN & CLEAN UP               remove attacker resources; purge from history; tell security/legal
4. PREVENT                          scanning, hooks, short-lived credentials
```

**Picture it.** You lose your house keys on a busy street. Changing your locks is the first job. Searching the street for the keys, or asking who saw them, comes after.

**The one thing to know.** **Deleting the commit does not un-leak the secret.** Forks, clones, caches and scanners already have it. **Rotate.**

**Try it now:** run `gitleaks detect` (or `trufflehog git file://.`) on one of your repos.

**You've got it if you can:** walk through the first hour after "AWS access key found in a public repo", in the right order.

> 📖 **That's the core.** Everything below is the **full reference**. Hands-on drill: [86 · AWS key pushed to a public repo](86-drill-aws-key-leaked.md). Real case: [75 · CircleCI 2023](75-circleci-2023.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Notifications | GitHub secret scanning alert; AWS email/support case "your access key was found publicly" (AWS may apply a quarantine policy to the exposed key); GitGuardian/partner alerts |
| Cloud bill / usage | Sudden EC2 instances in unusual regions (crypto-mining), GPU instances, unexpected spend (topic 43) |
| CloudTrail | API calls from unknown IPs/user agents using the key; `CreateUser`, `CreateAccessKey`, `RunInstances` in many regions |
| Other providers | Unexpected API usage on Stripe/Twilio/OpenAI/etc. accounts |
| Code review | `.env`, `credentials`, `terraform.tfstate`, `kubeconfig` files committed |

---

## Analogy

The **lost house keys** (Core).

---

## How it happens

| Path | Example |
|---|---|
| **Committed files** | `.env`, `config.yaml` with passwords, `~/.aws/credentials` copied into a repo, hard-coded tokens in code |
| **Terraform state** | State contains secrets in plain text; committed or stored in an open bucket ([Terraform 51](../../Terraform/notes/51-secrets.md)) |
| **CI logs** | `echo $TOKEN`, `set -x`, verbose tools printing headers; masking misses transformed values (base64) |
| **Container images** | `COPY .env` or `ARG SECRET` baked into a layer; anyone with the image can extract it ([Docker 18](../../Docker/notes/18-arg-env-labels-secrets.md)) |
| **Public by mistake** | A private repo made public; a gist; a pasted log in a support ticket or chat |
| **Compromised CI/vendor** | Secrets stored in a CI system that gets breached (CircleCI 2023, topic 75) |
| **Long-lived keys** | Static access keys valid for years; one leak = long-term access |

---

## How to find it

**First:** which secret, where, since when, how exposed (public or private repo, how many forks/clones)?

```bash
git log -p -S 'AKIA' --all | head -40           # when was the AWS key pattern introduced?
gitleaks detect --source . --log-opts="--all" --report-path leaks.json
```

**Second:** what can the secret do? (blast radius)

```bash
aws iam list-access-keys --user-name ci-deployer
aws iam list-attached-user-policies --user-name ci-deployer
aws iam list-user-policies --user-name ci-deployer
```

**Third:** was it used? By whom?

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAXXXXEXAMPLE \
  --start-time 2026-11-19T00:00:00Z --max-results 50 \
  --query 'Events[].[EventTime,EventName,AwsRegion,Username]' --output table
```
```
| 2026-11-20T03:04:11Z | GetCallerIdentity | us-east-1  | ci-deployer |
| 2026-11-20T03:04:40Z | DescribeRegions   | us-east-1  | ci-deployer |
| 2026-11-20T03:05:12Z | RunInstances      | ap-south-1 | ci-deployer |
| 2026-11-20T03:05:13Z | RunInstances      | sa-east-1  | ci-deployer |
```
- `GetCallerIdentity` from an unknown IP minutes after the push → someone tested the key. `RunInstances` in regions you never use → abuse (likely crypto-mining).
- ⚠️ `lookup-events` covers **management events for 90 days per region**: check **every** region (or use an organisation trail in a central bucket/CloudTrail Lake).

---

## How to fix it

### Immediate (in this order)

1. **Revoke / deactivate** the leaked credential:
   ```bash
   aws iam update-access-key --user-name ci-deployer --access-key-id AKIAXXXXEXAMPLE --status Inactive
   ```
   For other providers: revoke the token in their console; rotate DB passwords.
2. **Issue a new secret** and update the systems that legitimately use it (secret manager, CI). Expect a short disruption; plan it, but don't delay revocation for it.
3. **Check for persistence:** did the attacker create new users, keys, roles, Lambda functions, or EC2 key pairs? Remove them. Check **all regions**.
4. **Stop abuse:** terminate attacker instances, delete their resources, and contact the provider (AWS support can help with charges in some cases).
5. **Engage security/legal** if data access is possible (S3 `GetObject`, database access).

### Clean up the repository (after rotation)

- Remove the secret from the current code; for history, use `git filter-repo` (or BFG) and force-push; ask GitHub support to clear cached views if needed. Remember forks and clones still exist — that's why rotation comes first.

### Permanent

- **No long-lived cloud keys in CI**: use **OIDC** federation (GitHub Actions/GitLab/Jenkins → cloud role with short-lived tokens) ([GHA 36](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md)).
- **Secret managers** (AWS Secrets Manager, Vault, External Secrets Operator), not files.
- **Push protection** and **secret scanning** (GitHub secret scanning push protection, gitleaks in pre-commit and CI).
- `.gitignore` for `.env`, state files, kubeconfigs; state in encrypted remote backends.
- Least-privilege for every credential; automatic rotation where possible.
- BuildKit secret mounts for images (never `COPY .env`).

---

## How to verify the fix

- The old credential fails (`InvalidClientTokenId`); no new events from it in CloudTrail.
- Legit systems work with the new secret.
- No attacker-created resources remain (all regions, IAM).
- Scanners show the repo clean; push protection blocks a test secret.

---

## Prevention

- Alerts: GuardDuty (e.g. findings on credential misuse and crypto-mining), billing anomaly detection, CloudTrail alerts on `CreateUser`/`CreateAccessKey`.
- Regular audit of IAM users with access keys (aim for zero in CI).
- Developer training plus tooling (pre-commit) — tooling beats training.

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a developer committed a `.env` with a long-lived CI deploy key to a public repo. Root cause: long-lived keys existed at all, with broad permissions, and nothing blocked the push.

2. **Q: How did you detect it? Why not earlier?**
   A: GitHub secret scanning alert / AWS notification, minutes after push. Earlier = push protection blocking the commit.

3. **Q: What did you check first?**
   A: What the key could do (its policies), while revoking it — because blast radius decides urgency and who to involve.

4. **Q: How did you stop impact quickly?**
   A: Deactivated the key within minutes, then rotated and updated CI, then terminated instances launched by the attacker.

5. **Q: Permanent fix?**
   A: OIDC for CI (no static keys), push protection and pre-commit scanning, least privilege, GuardDuty and billing anomaly alerts.

6. **Q: How did you confirm?**
   A: Old key invalid, no further CloudTrail activity from it, no unknown IAM entities or resources in any region.

7. **Q: What if the attacker created their own access keys before you revoked yours?**
   A: Revoking yours doesn't stop them. Search CloudTrail for `CreateUser`, `CreateAccessKey`, `CreateRole`, `AttachRolePolicy`, `CreateLoginProfile`, and remove those. In serious cases, treat the account as compromised and follow the full incident response with the security team.

8. **Q: Why rotate before cleaning Git history?**
   A: The secret is already copied by bots, forks and clones; history cleaning can't recall it. Only rotation makes the leaked value useless.

9. **Q: What would you do differently?**
   A: Remove the possibility: no static cloud keys anywhere, push protection on all repos.

10. **Q: What if the leak was in a private repo?**
    A: Still rotate: many people, integrations and backups can read private repos, and repos can become public by mistake. Urgency is lower than public, but the process is the same.
