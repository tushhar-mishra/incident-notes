# 40 · `rm -rf` and the Wrong Environment

> **Level:** ★ foundation
>
> **You will learn:** why "**human error**" incidents are really **tooling and access** problems, the classic forms (`rm -rf` on the wrong path or host, `DROP` on the wrong database, `kubectl delete` in the wrong context, a script run against prod instead of staging), the shell mistakes that turn small slips into disasters (empty variables, globbing), how to **recover** (stop, preserve, restore), and the **guardrails** that make these mistakes hard: separate credentials, visible context, least privilege, dry runs, confirmations, and limits.
>
> **You need:** topics 01–39. Helpful: [Linux 007 · Files and directories](../../linux-mastery-rhel/notes/007-files-and-directories.md), [Linux 045 · Robust scripts](../../linux-mastery-rhel/notes/045-robust-scripts.md), [K8s 12 · kubectl contexts](../../CKA-Certification-Course-2025/notes/12-kubectl-kubeconfig-and-contexts.md), [K8s 57 · RBAC](../../CKA-Certification-Course-2025/notes/57-rbac.md).
>
> **Part 2 · Recurring failures · G. Human error, security, cost** · Previous: [39 · Third-party and region outages](39-third-party-and-region-outages.md) · Next: [41 · Terraform destroying production](41-terraform-destroying-production.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Engineers are tired, multitask and have many terminals open. Sooner or later someone runs the **right command** in the **wrong place**. GitLab 2017 (topic 46): an engineer deleted data on the **primary** database server while intending to clean up a replica. The fix is not "be careful"; it's making the wrong place **obvious** and the damage **limited**.

```
$ kubectl config current-context
prod-eu-1                    ← you thought it was staging
$ kubectl delete ns payments  # oops

$ DIR=""                     ← variable empty because a previous step failed
$ rm -rf "$DIR"/*            ← becomes rm -rf /*
```

**Picture it.** Two identical-looking doors in a hospital: one to the storage room, one to the operating theatre. The fix is not "nurses should be more careful"; it's painting one door red, adding a sign and a lock.

**The one thing to know.** Ask "**how did the system let one command do so much damage?**". The answers (shared credentials, invisible context, no confirmation, no limits) are what you fix.

**Try it now:** make your production terminals look different: red prompt, and the kube context and AWS profile always visible.

**You've got it if you can:** list five guardrails that would have prevented a wrong-environment deletion, and write a safe shell script header.

> 📖 **That's the core.** Everything below is the **full reference**. Real cases: [45 · AWS S3 2017](45-aws-s3-2017.md), [46 · GitLab 2017](46-gitlab-2017.md), [47 · Atlassian 2022](47-atlassian-2022.md).

---

## Symptoms

| Where | What you see |
|---|---|
| Users | Sudden total loss of a service or its data |
| Audit logs | A manual command by a human/role just before (CloudTrail `Delete*`/`Terminate*`, Kubernetes audit `delete`, database logs `DROP`) |
| Shell history | The command itself |
| Monitoring | Resources disappearing (pods, namespaces, volumes, instances), not just failing |
| Person | Often the engineer who ran it raises the alarm first — make sure your culture rewards that |

---

## Analogy

The **identical hospital doors** (Core).

---

## How it happens

| Mistake | How it happens | Guardrail |
|---|---|---|
| **Wrong host** | Two SSH sessions open; prod and replica look the same | Coloured prompts with hostname/role; banners; separate terminal profiles |
| **Wrong kube context** | `kubectl` uses the last context set | Show context in prompt (kube-ps1); separate kubeconfigs per env; `kubectl` wrappers that confirm for prod; read-only default roles |
| **Wrong cloud account** | `AWS_PROFILE` left on prod | Separate accounts per environment; prod access only via short-lived, elevated sessions |
| **Empty variable** | `rm -rf "$DIR/"*` with `DIR` unset | `set -u`; `${DIR:?}` (fails if empty); never `rm -rf` on computed paths without checks |
| **Glob surprises** | `rm -rf * .log` (space typo) | `rm -i` / `trash`; reviewed scripts; `--preserve-root` (GNU default) |
| **Script defaults to prod** | Default target value is production | Default to **nothing**; require explicit env argument |
| **Too much power** | Daily-use credentials can delete prod | Least privilege; break-glass for destructive actions |
| **Tool with no limits** | One command can remove 100% of something | Tools refuse beyond a % or count; rate-limited removal (AWS's fix after S3 2017) |

### Safe shell script header

```bash
#!/usr/bin/env bash
set -euo pipefail                # stop on error, on unset variables, on pipe failures
IFS=$'\n\t'

ENV="${1:?usage: $0 <env> }"     # fail if no env given
[[ "$ENV" =~ ^(dev|staging|prod)$ ]] || { echo "bad env"; exit 1; }
TARGET_DIR="/data/${ENV}/tmp"
[[ -d "$TARGET_DIR" ]] || { echo "no such dir"; exit 1; }

echo "About to delete files older than 7 days in: $TARGET_DIR"
find "$TARGET_DIR" -mindepth 1 -mtime +7 -print | head -20     # show a sample
read -r -p "Type the env name to confirm: " ok; [[ "$ok" == "$ENV" ]] || exit 1
find "${TARGET_DIR:?}" -mindepth 1 -mtime +7 -delete
```

---

## How to find it

**First:** what was destroyed, by whom, when? (Audit logs are the fastest way.)

```bash
# AWS
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteDBInstance --max-results 5
# Kubernetes (if audit logging is on; on EKS in CloudWatch Logs)
#   filter: verb="delete" and objectRef.resource="namespaces"
kubectl get events -A --sort-by=.lastTimestamp | tail -30
# Shell history of the session (if still open) / bastion session logs
```

**Second:** what's the current state? Is the damage **still happening** (a script still running)?

```bash
ps -ef | grep -E "rm -rf|find .* -delete" | grep -v grep
```

**Third:** what recovery options exist? Snapshots, backups, PITR, replicas, Git (for manifests), Terraform state (for infra), object versioning (S3), trash/soft-delete features.

---

## How to fix it

### Immediate

1. **Stop the damage:** `Ctrl-C`, kill the process, revoke the session — **before** anything else.
2. **Don't make it worse:** no more writes to the affected disk (deleted files might be recoverable); don't "try things" on the only remaining copy. Take a snapshot of what's left.
3. **Declare** an incident, even if you caused it. Fast honesty saves hours.
4. **Restore** from the best source:
   - Kubernetes objects: re-apply from Git (GitOps makes this fast: Argo CD resync).
   - Volumes/DBs: snapshots, PITR (topic 34).
   - Cloud resources: Terraform apply (infra), backups (data).
   - S3: object versioning, previous versions.

### Permanent

- Separate credentials and accounts per environment; production access via just-in-time elevation.
- Visible context everywhere (prompt colours, context in prompt, banners).
- **Deletion protection** on critical resources: RDS `DeletionProtection`, EC2 termination protection, S3 versioning + Object Lock, Terraform `prevent_destroy` (topic 41), Kubernetes admission policies blocking deletion of critical namespaces.
- Tools with **built-in limits** and **dry runs**; destructive operations via reviewed automation (pipelines), not ad-hoc shells.
- Two-person rule for high-risk manual operations (one types, one reviews).

---

## How to verify the fix

- Restored resources match expected state (Git/Terraform plan shows no diff; data checks pass).
- Users unaffected; no partial leftovers.
- Guardrails tested: the same command, run again in the same way, is now blocked or confirmed.

---

## Prevention

- Audit logging on everything (CloudTrail, Kubernetes audit, database audit) with alerts on mass deletions.
- Regular access reviews: who can delete what in prod?
- Practise restores of the things you'd lose (topic 34).

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Not the person. Example: one engineer's everyday credentials could delete the production namespace, the kube context wasn't visible, and there was no admission policy protecting critical namespaces. The command was the trigger; the lack of guardrails was the cause.

2. **Q: How did you detect it? Why not earlier?**
   A: The engineer noticed immediately and declared. Monitoring showed pods disappearing a minute later. Nothing prevented it upfront.

3. **Q: What did you check first?**
   A: That the deletion had stopped, then exactly what was removed (audit log), then which recovery source was newest.

4. **Q: How did you stop impact quickly?**
   A: Re-synced the namespace from Git with Argo CD; volumes were retained (Retain reclaim policy), so data reattached.

5. **Q: Permanent fix?**
   A: Read-only default access to prod, JIT elevation for writes, context in prompts, a Kyverno/Gatekeeper policy blocking deletion of labelled critical namespaces.

6. **Q: How did you confirm?**
   A: Argo CD showed Synced/Healthy; data checks passed; the same delete is now rejected by the admission policy.

7. **Q: What if the restore source was also gone?**
   A: Look for any other copy (replicas, warehouse, exports), recover what's possible, and be transparent with customers. Then: backups in a separate account with immutability.

8. **Q: How do you make sure people report their own mistakes?**
   A: Blameless culture (topic 11): thank them for speed, never punish honest mistakes, and visibly fix the system instead.

9. **Q: What would you do differently?**
   A: Make destructive actions go through pipelines with reviews and limits; keep humans' everyday access read-only.

10. **Q: What if this happened during an incident, when people are rushing?**
    A: That's when it's most likely. The IC enforces "announce before you act" and a second person checks the target before destructive commands.
