---
title: "C1 — Terraform State Issues"
description: "Diagnose and resolve Terraform state corruption, locking, and drift issues in AFT"
status: active
severity: CRITICAL
triggers: ["state lock error", "state corruption", "state file missing", "terraform state error", "DynamoDB lock"]
owner: devops-agent
objective: "Restore healthy Terraform state management for AFT operations"
context: "AFT stores Terraform state in S3 with DynamoDB locking. State issues can block all AFT operations. State corruption is the most dangerous failure mode as it can cause resource orphaning or duplicate creation."
---

## Phase 1 — Triage

MUST:
- Check for state lock in DynamoDB: `aws dynamodb scan --table-name aft-backend-<account-id>-<region> --filter-expression 'attribute_exists(LockID)'`
- Verify state file exists in S3: `aws s3 ls s3://aft-backend-<account-id>-<region>/`
- Check S3 versioning for state file history: `aws s3api list-object-versions --bucket aft-backend-<account-id>-<region> --prefix <state-key> --max-items 5`

SHOULD:
- Check if any CodeBuild projects are currently running that hold the lock
- Review the lock metadata for the process that holds it
- Verify S3 bucket versioning is enabled for state recovery

MAY:
- Compare current state with actual infrastructure using `terraform plan`
- Check for concurrent pipeline executions that may cause lock contention

## Phase 2 — Remediate

MUST:
- For stale locks: force unlock with the lock ID: `terraform force-unlock <lock-id>` (only after confirming no active process holds it)
- For corrupted state: restore from S3 versioning: `aws s3api get-object --bucket <bucket> --key <key> --version-id <version-id> state.tfstate`
- For missing state: DO NOT recreate — investigate why it was deleted

SHOULD:
- Verify the restored state matches actual infrastructure
- Run `terraform plan` after state restoration to check for drift
- Enable S3 bucket versioning if not already enabled

MAY:
- Use `terraform state list` to verify state contents
- Use `terraform import` to re-import resources missing from state

## Safety Ratings

```
safety_ratings:
  - "dynamodb scan: GREEN — read-only lock check"
  - "s3 ls: GREEN — read-only state file listing"
  - "terraform force-unlock: RED — removes state lock, risk of concurrent modification"
  - "s3api get-object (version restore): YELLOW — restores previous state version"
```

## Escalation Conditions

- State file is missing and S3 versioning was not enabled (no recovery possible)
- State corruption affects multiple AFT components simultaneously
- Force-unlock causes a concurrent modification conflict
- State drift is extensive and manual reconciliation is required

## Data Sensitivity

- Terraform state files: CRITICAL — contain all resource ARNs, configurations, and potentially sensitive outputs
- DynamoDB lock entries: LOW — lock metadata only
- S3 version history: HIGH — previous state versions contain historical resource data
- `terraform state list` output: MEDIUM — reveals managed resource inventory

## Prohibited Actions

- NEVER delete Terraform state files from S3
- NEVER force-unlock without confirming no active process holds the lock
- NEVER manually edit state files — use `terraform state` commands
- NEVER disable S3 versioning on the state bucket

## Phase 3 — Rollback

1. If force-unlock caused issues, check for any concurrent modifications and reconcile
2. If state was restored to wrong version, restore the correct version from S3 versioning
3. If `terraform import` added incorrect resources, use `terraform state rm` to remove them
4. Document all state manipulation operations with timestamps
5. Run `terraform plan` to verify state consistency after any changes

## Common Issues

- symptoms: "Error acquiring the state lock — ConditionalCheckFailedException"
  diagnosis: "A previous Terraform operation crashed without releasing the DynamoDB lock."
  resolution: "Verify no active builds: `aws codebuild list-builds-for-project --project-name <name>`. If none active, force-unlock."

- symptoms: "State file shows resources that no longer exist"
  diagnosis: "Resources were deleted outside of Terraform (manually or by another process)."
  resolution: "Run `terraform plan` to identify drift. Use `terraform state rm` for deleted resources."

## Output Format

```yaml
root_cause: "terraform_state — <specific_cause>"
evidence:
  - type: state_error
    content: "<lock details, state file status, and drift information>"
severity: CRITICAL
mitigation:
  immediate: "Restore state integrity"
  long_term: "Implement state backup automation and lock monitoring"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
