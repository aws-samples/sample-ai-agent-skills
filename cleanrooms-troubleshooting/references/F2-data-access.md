---
title: "F2 — Data Access Issues"
description: "Diagnose Clean Rooms data access problems including S3, Glue, and KMS access"
status: active
severity: HIGH
triggers:
  - "data access"
  - "s3 access clean rooms"
  - "glue access"
  - "kms access"
  - "data not accessible"
owner: devops-agent
objective: "Resolve data access issues for Clean Rooms table configuration and queries"
context: "Clean Rooms accesses data through a service role that needs S3, Glue, and optionally KMS permissions. Common issues include S3 bucket policy blocking access, Glue catalog permissions, KMS key access, and cross-account data access."
---

## Phase 1 — Triage

MUST:
- Verify service role has S3 access: `aws iam get-role-policy --role-name <role> --policy-name <policy>`
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Verify Glue table access: `aws glue get-table --database-name <db> --name <table>`
- If KMS encrypted, check key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`

SHOULD:
- Verify S3 bucket is in the same region as Clean Rooms
- Check if S3 bucket has Block Public Access (should be enabled)
- Verify Glue Data Catalog permissions

MAY:
- Test S3 access with the service role: assume role and list objects
- Check CloudTrail for S3 or KMS access denied events

## Phase 2 — Remediate

MUST:
- Grant service role S3 permissions: `s3:GetObject`, `s3:GetBucketLocation`, `s3:ListBucket`
- Grant Glue permissions: `glue:GetTable`, `glue:GetDatabase`, `glue:GetPartitions`
- If KMS encrypted, grant `kms:Decrypt` to the service role
- Ensure S3 bucket policy does not explicitly deny the service role

SHOULD:
- Use resource-based conditions to limit access to specific prefixes
- Verify data format is compatible (Parquet, CSV, JSON)

## Output Format

```yaml
root_cause: "data_access — <specific_cause>"
evidence:
  - type: service_role_permissions
    content: "<service role policy>"
  - type: s3_access
    content: "<S3 bucket policy>"
severity: HIGH
mitigation:
  immediate: "Fix service role permissions or bucket policy"
  long_term: "Use IaC to manage service roles and data access policies"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Grant service role S3 permissions | YELLOW |
| Grant Glue permissions | YELLOW |
| Grant kms:Decrypt to service role | YELLOW |
| Ensure S3 bucket policy does not deny service role | YELLOW |
| Use resource-based conditions to limit access | GREEN |

## Escalation Conditions

- Fix requires modifying production S3 bucket policies or KMS key policies
- Fix involves broadening data access permissions

## Data Sensitivity

- **HIGH**: data access configurations control which underlying datasets are exposed through Clean Rooms
- S3 and KMS permissions determine the scope of data available for collaborative queries

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If S3 permissions were modified, restore previous bucket policy and IAM role policy
2. If Glue permissions were changed, revert to previous policy
3. If KMS key access was granted, revoke if no longer needed
4. Verify rollback by confirming service role permissions match expected data access scope

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
