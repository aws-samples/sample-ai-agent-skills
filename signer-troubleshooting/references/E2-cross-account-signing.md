---
title: "E2 — Cross-Account Signing"
description: "Diagnose cross-account signing configuration and access issues"
status: active
severity: HIGH
triggers: ["cross-account signing failed", "cannot sign from another account", "sharing signing profile"]
owner: devops-agent
objective: "Resolve cross-account signing access and configuration issues"
context: "Cross-account signing requires a resource policy on the signing profile granting signer:StartSigningJob to the target account. The target account also needs IAM permissions and S3 access."
---

## Phase 1 — Triage

MUST:
- Check signing profile permissions: `aws signer get-signing-profile --profile-name <profile-name>`
- Verify the profile has a resource policy granting cross-account access
- Check the target account's IAM permissions for Signer

SHOULD:
- Verify S3 bucket access from both accounts (source and destination)
- Check if the signing profile ARN is correctly referenced in the target account

MAY:
- Review CloudTrail in both accounts for cross-account signing errors
- Check if SCPs restrict cross-account Signer operations

## Phase 2 — Remediate

MUST:
- Add resource policy to signing profile: `aws signer add-profile-permission --profile-name <name> --action signer:StartSigningJob --principal <target-account-id> --statement-id <id>`
- Ensure target account IAM role has `signer:StartSigningJob` permission with the profile ARN as resource
- Verify S3 bucket policies allow cross-account access for source and destination

SHOULD:
- Use specific account IDs in resource policies, not wildcards
- Verify the signing profile version ARN is used in code signing configs

MAY:
- Set up cross-account S3 access via bucket policies or IAM roles
- Implement cross-account signing in CI/CD pipelines

## Common Issues

- symptoms: "StartSigningJob from target account returns AccessDeniedException"
  diagnosis: "Signing profile missing resource policy for the target account."
  resolution: "Add profile permission: aws signer add-profile-permission --principal <target-account-id>."

- symptoms: "Cross-account signing job fails with S3 access error"
  diagnosis: "S3 bucket in the signing account not accessible from the target account."
  resolution: "Update S3 bucket policy to allow cross-account access or use S3 in the target account."

## Safety Ratings

```
safety_ratings:
  - "get-signing-profile: GREEN — read-only profile details"
  - "add-profile-permission: YELLOW — grants cross-account signing access"
  - "remove-profile-permission: YELLOW — revokes cross-account access"
  - "s3 put-bucket-policy: YELLOW — modifies cross-account S3 access"
```

## Escalation Conditions

- Cross-account access requires Organization-level policy changes
- Multiple accounts need signing access and individual permissions are impractical
- S3 cross-account access conflicts with data residency requirements
- Security team needs to review cross-account signing architecture

## Data Sensitivity

- Signing profile resource policy: MEDIUM — reveals cross-account trust relationships
- Target account IDs: MEDIUM — reveals organizational structure
- S3 bucket policies: MEDIUM — reveal cross-account data access
- Cross-account signing jobs: MEDIUM — reveal code deployment patterns

## Prohibited Actions

- NEVER grant signing access to untrusted or unknown AWS accounts
- NEVER use wildcard (*) principals in signing profile resource policies
- NEVER share signing profile ARNs publicly
- NEVER configure cross-account S3 access without encryption requirements

## Phase 3 — Rollback

1. Remove cross-account permission: `aws signer remove-profile-permission --profile-name <name> --statement-id <id>`
2. Revert S3 bucket policy changes
3. Verify the target account can no longer sign with the profile
4. Update code signing configs in the target account if needed
5. Document the cross-account access changes

## Output Format

```yaml
root_cause: "cross_account_signing — <specific_cause>"
evidence:
  - type: cross_account_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix cross-account signing access"
  long_term: "Implement centralized signing account pattern"
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
