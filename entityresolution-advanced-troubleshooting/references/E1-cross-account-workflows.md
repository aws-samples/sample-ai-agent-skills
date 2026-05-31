---
title: "E1 — Cross-Account Workflows"
description: "Diagnose cross-account Entity Resolution workflow failures including resource policies, IAM roles, and data access"
status: active
severity: HIGH
triggers: ["cross-account workflow failed", "cannot access other account data", "cross-account permission denied", "multi-account entity resolution"]
owner: devops-agent
objective: "Resolve cross-account workflow access and execution failures"
context: "Cross-account workflows require resource policies on Glue databases, S3 buckets, and KMS keys in source accounts. The service role must have cross-account access. RAM may be needed for Glue catalog sharing."
---

## Phase 1 — Triage

MUST:
- Check the workflow configuration for cross-account resource references: `aws entityresolution get-matching-workflow --workflow-name <workflow-name>`
- Verify the service role can access cross-account resources: `aws iam get-role --role-name <role-name>`
- Check Glue resource policy in the source account: `aws glue get-resource-policy`

SHOULD:
- Verify S3 bucket policy in the source account allows cross-account access
- Check KMS key policy if data is encrypted
- Review RAM resource shares if using Glue catalog sharing: `aws ram list-resources --resource-owner SELF`

MAY:
- Test cross-account access with sts:AssumeRole
- Check CloudTrail in both accounts for access denied events
- Verify Organizations policies do not block cross-account access

## Phase 2 — Remediate

MUST:
- Update Glue resource policy to allow cross-account access
- Update S3 bucket policy to allow the service role from the other account
- Update KMS key policy to allow cross-account decryption

SHOULD:
- Use RAM for Glue catalog sharing when possible
- Create dedicated cross-account IAM roles with least-privilege
- Test cross-account access before running workflows

MAY:
- Set up Organizations SCPs to govern cross-account data access
- Implement centralized data catalog for multi-account environments

## Safety Ratings

```
safety_ratings:
  - "get-matching-workflow: GREEN — read-only workflow configuration"
  - "glue get-resource-policy: GREEN — read-only Glue policy"
  - "ram list-resources: GREEN — read-only RAM resource listing"
  - "glue put-resource-policy: RED — modifies Glue catalog access policy"
  - "s3api put-bucket-policy: RED — modifies S3 bucket access"
```

## Escalation Conditions

- Cross-account access requires source account admin approval
- KMS key policy changes affect other services using the same key
- Organizations SCPs block required cross-account access
- Compliance requirements restrict cross-account data sharing

## Data Sensitivity

- `aws entityresolution get-matching-workflow`: LOW — workflow configuration
- `aws glue get-resource-policy`: MEDIUM — reveals access policies
- `aws ram list-resources`: LOW — returns shared resource metadata
- Cross-account data: HIGH — contains PII from source account

## Prohibited Actions

- NEVER use wildcard principals in cross-account resource policies
- NEVER share KMS keys without understanding all data encrypted with them
- NEVER bypass Organizations SCPs for cross-account access
- NEVER access data in other accounts without data owner authorization

## Phase 3 — Rollback

1. If resource policies were modified incorrectly, restore previous policies
2. If RAM shares were created in error, revoke them
3. If KMS key policy was changed, restore previous configuration
4. Verify other services using the same resources are not affected
5. Document the cross-account configuration change

## Common Issues

- symptoms: "Workflow fails with AccessDeniedException for Glue table in another account"
  diagnosis: "The Glue resource policy in the source account does not allow cross-account access."
  resolution: "Update Glue resource policy to allow the Entity Resolution service role from the workflow account."

- symptoms: "Can access Glue table but cannot read S3 data"
  diagnosis: "The S3 bucket policy does not allow cross-account access, or KMS key policy blocks decryption."
  resolution: "Update S3 bucket policy and KMS key policy to allow the service role from the workflow account."

- symptoms: "RAM share exists but Glue table not visible"
  diagnosis: "RAM share invitation not accepted, or the share does not include the specific database/table."
  resolution: "Accept the RAM share invitation. Verify the share includes the required Glue resources."

## Output Format

```yaml
root_cause: "cross_account — <specific_cause>"
evidence:
  - type: cross_account_error
    content: "<resource policy and access details>"
severity: HIGH
mitigation:
  immediate: "Fix cross-account resource policies"
  long_term: "Implement centralized data access governance"
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
