# Policy-as-Code — Prevention, Defense-in-Depth & Real-World Use Cases

## Part 1 — Prevention (Pipeline Layer)

**Problem**: relying on PR review alone to catch bad infra (e.g., public S3 bucket) fails — reviewers miss things in large diffs, don't always know which attribute combos are dangerous, or are rushed. Review is a *should-catch*, not a *can't-bypass*.

**Fix**: Policy-as-Code — Sentinel (Terraform Cloud/Enterprise) or OPA (anywhere) — evaluates the Terraform plan as a hard gate before `apply`. Fail the policy, pipeline stops, `apply` never runs, regardless of who approved the PR.

**Pipeline shape**:
```
fmt → validate → plan → [POLICY CHECK] → manual approval → apply
```

**Example — OPA rule blocking public S3 buckets**:
```rego
package main

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket_public_access_block"
  resource.change.after.block_public_acls == false
  msg := sprintf("%s allows public ACLs — blocked", [resource.address])
}
```
```bash
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
conftest test plan.json -p policies/
```
If `conftest` fails, the GitHub Actions `apply` step (gated with `if: success()`) never executes.

**Limitation to know**: this only guards the pipeline path. Console access, a stray `aws s3api` CLI call, or a different tool entirely bypasses OPA completely — it never sees those actions.

## Part 2 — Defense Beyond the Pipeline (AWS-Native Layers)

Since the pipeline only covers one path, back it with enforcement inside AWS itself — independent of Terraform or any CI tool.

**Layer 2 — S3 Block Public Access, account level** (overrides any bucket ACL/policy trying to go public, however it was set):
```bash
aws s3control put-public-access-block \
  --account-id 123456789012 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

**Layer 3 — SCP (org level, if using AWS Organizations)** — evaluated *before* IAM, caps even admins/root:
```json
{
  "Effect": "Deny",
  "Action": ["s3:PutAccountPublicAccessBlock"],
  "Resource": "*",
  "Condition": {"StringEquals": {"s3:PutAccountPublicAccessBlock": "false"}}
}
```

**Layer 4 — AWS Config + auto-remediation** — continuous scan, not just at creation time:
- Managed rule: `s3-bucket-public-read-prohibited`
- SSM Automation document attached → auto-fixes the violation (re-blocks public access) within minutes, no human needed

**Full stack, in order of "what still catches it"**:
```
OPA/Sentinel in pipeline (before apply)
  → S3 Block Public Access, account level (blocks it even if apply ran)
    → SCP (blocks it even below IAM, even for admins)
      → AWS Config + auto-remediation (fixes it even if everything above failed)
```

**One-liner**: "Pipeline policy is one layer, not the whole defense — I'd back it with account-level S3 Block Public Access, an SCP so even admins can't override it, and Config auto-remediation as a last-resort safety net. Defense-in-depth, not a single point of enforcement."

## Part 3 — Real-World Policy-as-Code Use Cases (Beyond S3)

| Domain | Tool | Real-time example rule |
|---|---|---|
| **Terraform infra** | Sentinel / OPA | No public S3 buckets; no security group open to `0.0.0.0/0` on port 22; every resource must have `CostCenter` tag |
| **Kubernetes admission control** | OPA Gatekeeper / Kyverno | No container may run as root; every pod must set CPU/memory limits; no `:latest` image tag allowed in prod namespace |
| **Cloud account guardrails** | AWS SCP, Azure Policy, GCP Org Policy | Deny creation of resources outside approved regions; deny disabling of CloudTrail; deny EC2 instance types above a size threshold without approval tag |
| **CI/CD image security** | OPA + image scanners (Trivy/Grype) | No image with critical CVEs may push to prod ECR; base image must come from an approved internal registry |
| **Network/firewall config** | OPA on Terraform network modules | No NACL/security group rule allowing unrestricted inbound on database ports (3306, 5432) |
| **Cost governance** | OPA / Sentinel | Block `apply` if estimated monthly cost (via Infracost) exceeds a threshold without an approval tag |
| **Secrets hygiene** | OPA / Checkov | Deny any resource with a hardcoded secret-looking string in plan output (e.g., `password = "..."` literal instead of a `data` source reference) |

**Why this table matters for interviews**: naming 2-3 of these beyond just "S3 buckets" is what shows you understand PaC as a general discipline — one engine (OPA), same evaluation pattern, applied wherever a plan/manifest/config can be inspected before it takes effect — not a single Terraform trick.
