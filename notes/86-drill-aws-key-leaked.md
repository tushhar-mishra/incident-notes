# 86 · Drill: AWS Key Pushed to a Public Repo

> **Level:** ★★ intermediate
>
> **You will practise:** the security incident in the right order: **revoke** first, **investigate** in CloudTrail across **all regions**, **remove** attacker persistence and resources, **rotate** legitimate use, **clean** history, and prevent it (OIDC, push protection).
>
> **You need:** topics 42, 43, 75, 76. Helpful: [GHA 36 · OIDC](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md).
>
> **Part 4 · On-call drills** · Previous: [85 · Terraform plan destroys the prod DB](85-drill-terraform-destroy-prod-db.md) · Next: [87 · Cloud bill 5× overnight](87-drill-cloud-bill-5x.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **02:12 — Email + GitHub alert:** `Secret scanning: AWS Access Key ID AKIA…7XQ detected in acme/deploy-tools (public), commit 9f2c1e, pushed 02:04`

**The idea.** Assume the key is already being used. **Deactivate it now**, then find out what happened.

```
0–5 min    deactivate the key; declare a SEV1 security incident; involve security
5–30 min   CloudTrail (ALL regions): what did the key do? new users/keys/roles? instances?
30–60 min  remove attacker resources & persistence; rotate legit use (move to OIDC)
after      clean Git history; postmortem; push protection + OIDC everywhere
```

**The one thing to know.** **Deleting the commit doesn't help.** Rotation/revocation does.

> 📖 Read only **Symptoms**, then write your first five actions in order.

---

## Symptoms

- Secret scanning / AWS / partner alert about an exposed key.
- Possibly: CloudTrail activity from unknown IPs, EC2 instances in regions you never use, a cost anomaly (topic 43), GuardDuty findings.

## Analogy

Your house keys were posted on a public noticeboard. Change the locks first; then check whether anyone came in and what they took; then take the notice down.

## How it happens

A `.env` or script with an access key committed and pushed to a public repo; bots scan public pushes continuously. The key belongs to an IAM user for CI deploys with broad permissions (topic 42).

### Lab setup (safe)

Use a **sandbox AWS account** with a **budget alert**, or do this drill as a **tabletop**. In a sandbox: create an IAM user with a tightly scoped policy, create a key, commit it to a **private** test repo with secret scanning enabled, and practise the response. Never publish a real key.

## How to find it

**First — what can the key do?**

```bash
aws iam list-access-keys --user-name ci-deployer
aws iam list-attached-user-policies --user-name ci-deployer
aws iam list-groups-for-user --user-name ci-deployer
```

**Second — what did it do? (every region)**

```bash
for r in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo "== $r"
  aws cloudtrail lookup-events --region $r \
    --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIA...7XQ \
    --start-time 2026-11-20T02:00:00Z --max-results 50 \
    --query 'Events[].[EventTime,EventName,Username]' --output text
done
```
```
== us-east-1
2026-11-20T02:05:40Z  GetCallerIdentity
2026-11-20T02:06:02Z  CreateUser
2026-11-20T02:06:05Z  CreateAccessKey            ← persistence!
== ap-southeast-1
2026-11-20T02:07:11Z  RunInstances               ← likely crypto-mining
```
(If you have an organisation trail in S3 / CloudTrail Lake, query that instead — faster and complete.)

**Third — find attacker-created resources everywhere:**

```bash
aws iam list-users --query 'Users[?CreateDate>=`2026-11-20`].[UserName,CreateDate]'
for r in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  aws ec2 describe-instances --region $r --filters Name=instance-state-name,Values=running \
    --query 'Reservations[].Instances[].[InstanceId,InstanceType,LaunchTime]' --output text | sed "s/^/$r /"
done
```

## How to fix it

### Immediate (in this order)

```bash
aws iam update-access-key --user-name ci-deployer --access-key-id AKIA...7XQ --status Inactive
# attacker persistence:
aws iam list-access-keys --user-name backup-svc   # the user the attacker created
aws iam update-access-key --user-name backup-svc --access-key-id AKIA...ZZZ --status Inactive
aws iam delete-login-profile --user-name backup-svc 2>/dev/null
# then detach policies / delete the user after evidence is captured
aws ec2 terminate-instances --region ap-southeast-1 --instance-ids i-0abc...
```
- Capture evidence first (CloudTrail exports, instance details) for the security team.
- Restore legitimate CI: switch the pipeline to **OIDC** (fast path: a new scoped key stored in the secrets manager, temporarily).
- Open an AWS support case (unexpected charges, help with compromise).

### Permanent

- **No static keys for CI** — OIDC roles with branch/environment conditions.
- GitHub **push protection** + gitleaks pre-commit.
- SCPs: deny unused regions; deny `iam:CreateUser`/`CreateAccessKey` outside an admin path.
- GuardDuty, budgets and anomaly detection on every account.
- Clean history (`git filter-repo`) **after** rotation.

## How to verify the fix

- Old key: `InvalidClientTokenId`; no new CloudTrail events from it or from attacker-created principals.
- No unknown users, keys, roles or instances in any region.
- CI deploys work with the new credential path.

## Prevention

OIDC everywhere, push protection, least privilege, SCP region restrictions, security alerting (GuardDuty, CloudTrail alarms on IAM changes).

## Follow-up questions

1. **Q: Root cause?**
   A: A long-lived, over-privileged CI access key existed and could be committed; nothing blocked the push. The commit was the trigger.
2. **Q: How did you detect it? Why not earlier?**
   A: Secret scanning within minutes of the push. Push protection would have blocked it before it was public.
3. **Q: What did you check first?**
   A: Deactivated the key immediately, then what the key could do and did (CloudTrail, all regions).
4. **Q: How did you stop impact?**
   A: Deactivated keys (including attacker-created ones), terminated attacker instances, removed persistence.
5. **Q: Permanent fix?**
   A: OIDC for CI, push protection, SCPs, GuardDuty and budgets.
6. **Q: How did you confirm?**
   A: No activity from old or attacker credentials; no unknown resources; CI healthy.
7. **Q: What if the attacker had created a role with a trust policy to their own account?**
   A: Check `CreateRole`/`UpdateAssumeRolePolicy` events; remove or fix trust policies; review all roles' trust relationships. Consider the account compromised and follow full security IR.
8. **Q: How would you prevent it?**
   A: Make static keys unnecessary (OIDC) and impossible to push (push protection).
9. **Q: What would you do differently?**
   A: Have this exact runbook ready, including the all-regions loops, before it happens.
10. **Q: What if the key had read access to customer data in S3?**
    A: Check S3 data events (if enabled) or server access logs for `GetObject` by the key; involve legal/security for breach notification obligations.
