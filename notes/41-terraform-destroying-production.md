# 41 · Terraform Destroying Production

> **Level:** ★★ intermediate
>
> **You will learn:** the ways **Terraform** (and other IaC) can delete or replace production resources: **replace plans** (`-/+`) from a changed "force new" attribute, renamed resources and **`count` index shifts**, **wrong workspace / wrong state / wrong account**, **state loss**, `terraform destroy` or `-target` misuse, **drift** "fixes" that revert manual emergency changes, and **provider upgrades** that change defaults; how to **read a plan defensively**, and the guardrails: `prevent_destroy`, deletion protection, policy as code, plan review in CI, and `moved` blocks.
>
> **You need:** topics 01–40. Helpful: [Terraform 13 · Inside plan and apply](../../Terraform/notes/13-inside-plan-and-apply.md), [Terraform 21 · count](../../Terraform/notes/21-count.md), [Terraform 22 · for_each](../../Terraform/notes/22-for-each.md), [Terraform 24 · lifecycle](../../Terraform/notes/24-lifecycle.md), [Terraform 40 · moved and removed](../../Terraform/notes/40-moved-and-removed.md), [Terraform 42 · State recovery](../../Terraform/notes/42-state-recovery.md), [Terraform 54 · Policy as code](../../Terraform/notes/54-policy-as-code.md), [Terraform 66 · Production scenarios](../../Terraform/notes/66-production-scenarios.md).
>
> **Part 2 · Recurring failures · G. Human error, security, cost** · Previous: [40 · rm -rf and the wrong environment](40-rm-rf-and-wrong-environment.md) · Next: [42 · Secrets leaked in Git](42-secrets-leaked-in-git.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform does exactly what the **plan** says. Production gets destroyed when nobody **reads** (or understands) a plan that says "destroy" or "replace".

```
  # aws_db_instance.orders must be replaced
-/+ resource "aws_db_instance" "orders" {
      ~ identifier = "orders-prod" -> "orders-prod-v2" # forces replacement
    }
Plan: 1 to add, 0 to change, 1 to destroy.     ← the line that should stop everyone
```

**Picture it.** A builder's instructions: "Change the door colour" is fine. "Demolish the house and build a new one with a different door" is a very different job, even if the drawing looks almost the same.

**The one thing to know.** In any plan touching production, search for **`destroy`**, **`must be replaced`** and **`-/+`** before anything else. If you see them on a stateful resource (database, volume, bucket, DNS zone, load balancer), **stop**.

**Try it now:** in a lab, change the `identifier` of a resource that "forces replacement" and read the plan.

**You've got it if you can:** explain three ways a plan can destroy prod unexpectedly and the guardrail for each.

> 📖 **That's the core.** Everything below is the **full reference**. Your Terraform notes cover the mechanics; this topic is the incident view.

---

## Symptoms

| Where | What you see |
|---|---|
| Plan output | `Plan: N to add, M to change, K to destroy` with K > 0 on prod; `# ... must be replaced`; `-/+` |
| Apply output | `Destroying... aws_db_instance.orders`; `Destruction complete after 6m` |
| Monitoring | Database/LB/cluster gone; DNS records missing; users down |
| Pipeline | An auto-apply job ran on merge without a human reading the plan |
| State | `terraform state list` missing resources (state lost or wrong backend) → next plan wants to **create** duplicates, or someone "cleans up" |

---

## Analogy

The **builder's instructions** (Core).

---

## How it happens

| Cause | What the plan shows | Guardrail |
|---|---|---|
| **Force-new attribute changed** (DB identifier, subnet group, AZ, engine for some resources, user_data on some instance setups) | `must be replaced`, `forces replacement` | `prevent_destroy`; review; `create_before_destroy` where safe |
| **Resource renamed / moved into a module** | Destroy old address + create new | `moved {}` blocks ([Terraform 40](../../Terraform/notes/40-moved-and-removed.md)) |
| **`count` index shift** (remove item 0 from a list) | Items 1..n destroyed and recreated with new indexes | Use `for_each` with stable keys ([Terraform 21](../../Terraform/notes/21-count.md)) |
| **Wrong workspace / backend / account** | Plan wants to destroy "everything" or create everything | Separate state per env, account guard in provider (`allowed_account_ids`), visible context in CI |
| **State lost or replaced** | Plan wants to create everything that already exists; or someone runs `destroy` to "reset" | Remote state with versioning + locking; restore state, don't recreate |
| **`terraform destroy` / `-target`** run locally against prod | Exactly what was asked | No human prod credentials for apply; pipelines only |
| **Reverting an emergency manual fix** | Plan "fixes drift" by undoing what on-call did during an incident | Update code after manual changes (or `ignore_changes` temporarily) |
| **Provider upgrade** | New defaults → changes/replacements nobody intended | Pin providers; read upgrade guides; plan in staging first |
| **Auto-apply on merge** | Nobody looked at the plan | Manual approval for prod, policy checks fail on destroys |

---

## How to find it

**First (before apply):** read the plan for destroys and replacements.

```bash
terraform plan -out=tfplan
terraform show -json tfplan | jq -r '
  .resource_changes[]
  | select(.change.actions | index("delete"))
  | "\(.change.actions | join(",")) \(.address)"'
```
```
delete,create aws_db_instance.orders
delete aws_route53_record.api
```
- `delete,create` = replacement of the **database** → stop.

Why is it being replaced?
```bash
terraform plan -no-color | grep -B2 -A2 "forces replacement"
```

**Second (after damage):** what did the apply do?

```bash
# CI logs of the apply job; state history
terraform state list | wc -l
aws s3api list-object-versions --bucket tf-state-prod --prefix network/terraform.tfstate --max-items 5
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteDBInstance --max-results 5
```

**Third:** recovery sources: final snapshots (RDS creates one on delete unless `skip_final_snapshot = true`), automated backups retention after deletion (depends on settings), S3 versioning, previous state versions.

---

## How to fix it

### Immediate

1. **Stop the pipeline** (cancel the run, disable auto-apply) so nothing else gets destroyed.
2. **Restore the data** first (final snapshot / PITR / backup) into a **new** resource.
3. **Reconcile Terraform with reality:** `terraform import` the restored resource (or `import {}` blocks), or restore the previous state version if the state is the problem. Make sure the next plan shows **no further destroys**.
4. Update DNS/config to point to restored resources if identifiers changed.

### Permanent

```hcl
resource "aws_db_instance" "orders" {
  # ...
  deletion_protection       = true      # cloud-side guard: API refuses delete
  skip_final_snapshot       = false
  final_snapshot_identifier = "orders-prod-final"
  lifecycle {
    prevent_destroy = true               # Terraform refuses any plan that destroys this
  }
}

provider "aws" {
  allowed_account_ids = ["111111111111"]  # refuse to run against the wrong account
}
```

- **Policy as code** in CI (OPA/Conftest, Sentinel, Checkov): fail the plan if a stateful resource is deleted/replaced without an explicit override label ([Terraform 54](../../Terraform/notes/54-policy-as-code.md)).
- **Plan review** shown in the PR, **manual approval** for prod, `-target` banned in pipelines.
- `for_each` with stable keys; `moved` blocks for refactors.
- Small state files (blast radius) per component/environment ([SD 36](../../System-Design/notes/36-iac-at-scale.md)).
- Remote state with versioning, locking and restricted access.

---

## How to verify the fix

- `terraform plan` on prod → `No changes. Your infrastructure matches the configuration.`
- Restored resources healthy; data verified.
- A test PR that tries to replace the DB is **blocked** by policy/`prevent_destroy`.

---

## Prevention

- CI comments a **summary** of destroys/replacements at the top of each PR.
- Alerts on CloudTrail delete events for critical resource types.
- Regular "plan only" drift detection jobs (no auto-apply) ([Terraform 37 · Drift](../../Terraform/notes/37-drift.md)).

---

## Follow-up questions

1. **Q: What was the real root cause?**
   A: Example: a refactor moved the DB into a module without a `moved` block, so the plan was "destroy old, create new". The pipeline auto-applied on merge and the plan's "1 to destroy" wasn't highlighted. Root cause: no destroy guardrails (no `prevent_destroy`, no policy check, no manual approval).

2. **Q: How did you detect it? Why not earlier?**
   A: Database connection errors after the apply. It could have been caught at plan time by a policy check.

3. **Q: What did you check first?**
   A: That the pipeline was stopped, then what exactly the apply destroyed (apply log + CloudTrail), then the newest restore point.

4. **Q: How did you stop impact quickly?**
   A: Restored from the final snapshot to a new instance, pointed the app at it, then imported it into state.

5. **Q: Permanent fix?**
   A: `prevent_destroy` + `deletion_protection`, OPA policy failing destroys of stateful resources, manual prod approval, and `moved` blocks for refactors.

6. **Q: How did you confirm?**
   A: Plan shows no changes; a test PR replacing the DB is rejected.

7. **Q: What if the restore failed (no final snapshot)?**
   A: Use automated backups/PITR if retained, cross-region/cross-account backup copies, replicas, or exports. If nothing, recover what's possible from other systems and communicate. Then: `skip_final_snapshot = false` everywhere, backup vaults outside the account.

8. **Q: Someone made an emergency manual change during an incident; how do you stop Terraform reverting it?**
   A: Record it in the incident channel, and immediately follow up with a code change (or a temporary `ignore_changes`) before the next apply. Drift checks should flag it for the owner.

9. **Q: What would you do differently?**
   A: Treat the plan as the change to review, with destroys highlighted and blocked by default on prod.

10. **Q: What if the wrong state file was used (staging state against prod account)?**
    A: The plan would want to create/destroy unexpected things. `allowed_account_ids` and separate backends per account prevent it; if it already happened, stop, restore the correct state version, and import anything created by mistake.
