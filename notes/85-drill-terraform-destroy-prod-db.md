# 85 · Drill: Terraform Plan Wants to Destroy the Prod Database

> **Level:** ★★ intermediate
>
> **You will practise:** the incident you **prevent**: a pipeline's plan for a harmless-looking PR shows `-/+` (replace) on the production database. Stopping the pipeline, finding **why** Terraform wants to replace it, fixing it **without** destroying anything (`moved` blocks, reverting a force-new change, state fixes), and adding guardrails.
>
> **You need:** topics 41, 76. Helpful: [Terraform 13](../../Terraform/notes/13-inside-plan-and-apply.md), [Terraform 24 · lifecycle](../../Terraform/notes/24-lifecycle.md), [Terraform 40 · moved/removed](../../Terraform/notes/40-moved-and-removed.md), [Terraform 66 · Production scenarios](../../Terraform/notes/66-production-scenarios.md).
>
> **Part 4 · On-call drills** · Previous: [84 · Queue backlog and retries](84-drill-queue-backlog-retries.md) · Next: [86 · AWS key pushed to a public repo](86-drill-aws-key-leaked.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

> 📟 **10:15 — Slack from the platform bot:** `PR #482 "refactor: move db into module" — plan: 2 to add, 0 to change, 2 to destroy (prod). Auto-apply on merge is ON. Merge button is green.`

**The idea.** Nothing is broken **yet**. Your job is to make sure it stays that way: **stop the merge/apply**, read the plan, find the cause, and change the code so the plan shows **no destroys**.

```
STOP: block the merge / disable auto-apply for prod
READ: which resources are destroyed/replaced, and WHY (forces replacement? address changed?)
FIX:  moved {} block (refactor) / revert force-new attribute / import / state mv
VERIFY: plan shows 0 to destroy (ideally "No changes")
GUARD: prevent_destroy, deletion_protection, policy check, manual approval for prod
```

**The one thing to know.** **Never "apply and see"** with a destroy on a stateful resource. Read every `must be replaced` line.

> 📖 Read only **Symptoms**, then write what you do in the first five minutes.

---

## Symptoms

```
  # aws_db_instance.orders will be destroyed
  # (because aws_db_instance.orders is not in configuration)
  - resource "aws_db_instance" "orders" { ... }

  # module.database.aws_db_instance.orders will be created
  + resource "aws_db_instance" "orders" { ... }

Plan: 2 to add, 0 to change, 2 to destroy.
```

## Analogy

The moving company's plan says "demolish the house, build a new one with the same furniture" when you only asked to relabel the boxes. You don't sign the plan; you fix the instructions.

## How it happens

Topic 41: renaming or moving a resource (into a module) changes its **address**; Terraform sees "old address gone → destroy" and "new address → create" unless told they're the same object (`moved`). Other causes: a force-new attribute changed (identifier, subnet group, engine), `count` index shifts, wrong workspace/state.

### Lab setup (break it, safely with a local provider)

```hcl
# main.tf (v1)
resource "random_pet" "db" { length = 2 }    # stands in for a database
```
```bash
terraform init && terraform apply -auto-approve
```
```hcl
# v2: "refactor" into a module
module "database" { source = "./db" }       # ./db/main.tf contains: resource "random_pet" "db" { length = 2 }
```
```bash
terraform plan        # → 1 to add, 1 to destroy
```

## How to find it

**First — stop the risk:** request changes on the PR / disable auto-merge; if a pipeline is already running, **cancel** it (Jenkins abort, GitHub Actions cancel).

**Second — list destroys/replacements and the reason:**

```bash
terraform plan -out=tfplan
terraform show -json tfplan | jq -r '.resource_changes[] | select(.change.actions|index("delete")) | "\(.change.actions|join(",")) \(.address)"'
terraform plan -no-color | grep -E "will be destroyed|must be replaced|forces replacement|not in configuration"
```
- `not in configuration` + a matching `will be created` at a new address → **address change** (refactor).
- `forces replacement` next to an attribute → **force-new change**.

**Third — check you're on the right state/account:** `terraform workspace show`, backend config, `aws sts get-caller-identity`.

## How to fix it

### Immediate (no apply needed)

- **Refactor:** add a `moved` block, then re-plan:
  ```hcl
  moved {
    from = aws_db_instance.orders
    to   = module.database.aws_db_instance.orders
  }
  ```
- **Force-new attribute:** revert that attribute, or plan a proper migration (snapshot + restore + cutover) if the change is truly needed.
- **count index shift:** switch to `for_each` with stable keys plus `moved` blocks.
- **Wrong state/workspace:** stop; point at the right backend; never "fix" by applying.

Re-plan until: `Plan: 0 to add, 0 to change, 0 to destroy` (or only intended changes).

### Permanent

```hcl
resource "aws_db_instance" "orders" {
  deletion_protection = true
  skip_final_snapshot = false
  lifecycle { prevent_destroy = true }
}
```
- Policy as code in CI: fail plans that delete/replace stateful resource types unless an override label is present.
- Manual approval for prod applies; PR comment summarising destroys at the top.
- `allowed_account_ids` in the provider block.

## How to verify the fix

- Plan shows no destroys; PR re-reviewed; merge applies cleanly with "0 to destroy".
- A test PR that would destroy the DB is **blocked** by `prevent_destroy` / policy.

## Prevention

Guardrails above, `moved` blocks as standard practice for refactors, plan review checklists, drift detection jobs.

## Follow-up questions

1. **Q: Root cause (if it had been applied)?**
   A: A refactor changed the resource address without a `moved` block, and the pipeline auto-applied without a destroy guard. The human didn't do anything unusual; the system allowed it.
2. **Q: How was it detected?**
   A: By reading the plan in the PR before merge — the cheapest possible detection.
3. **Q: What did you check first?**
   A: Stopping the apply path, then which resources and why (address change vs force-new).
4. **Q: How did you stop impact?**
   A: Blocked merge/auto-apply; the fix was code-only.
5. **Q: Permanent fix?**
   A: `prevent_destroy`, deletion protection, policy checks, manual prod approval, `moved` blocks.
6. **Q: How did you confirm?**
   A: Plan shows no destroys; a deliberate destructive test PR is rejected.
7. **Q: What if it had already been applied?**
   A: Topic 41: stop the pipeline, restore from the final snapshot/PITR into a new instance, import it, update consumers, make the plan clean.
8. **Q: How would you prevent it?**
   A: Destroys on stateful resources should be impossible without a deliberate override.
9. **Q: What would you do differently?**
   A: Disable auto-apply for prod; require plan approval by a second person.
10. **Q: What if the plan wanted to replace 50 resources because of a provider upgrade?**
    A: Pin the provider back, read the upgrade guide, and handle changes in a dedicated PR in staging first (Terraform 62).
