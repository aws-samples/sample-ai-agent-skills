---
title: "A1 — Datastore Creation"
description: "Diagnose datastore creation failures"
status: active
severity: HIGH
triggers: ["cannot create datastore", "datastore creation failed", "CreateDatastore error"]
owner: devops-agent
objective: "Resolve HealthImaging datastore creation issues"
context: "Datastores are isolated containers for medical imaging data. Encryption is set at creation and cannot be changed."
---

## Phase 1 — Triage

MUST:
- Check existing datastores: `aws medical-imaging list-datastores`
- Verify IAM permissions for medical-imaging:CreateDatastore
- Check service quotas: `aws service-quotas list-service-quotas --service-code medical-imaging`

SHOULD:
- Verify KMS key permissions if using customer-managed key: `aws kms describe-key --key-id <key-id>`
- Check if account has reached datastore limit in the region

MAY:
- Review CloudTrail for CreateDatastore errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medical-imaging.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions: ensure medical-imaging:CreateDatastore is allowed
- For customer-managed KMS: grant kms:CreateGrant, kms:DescribeKey to the HealthImaging service
- Request quota increase if datastore limit reached

SHOULD:
- Use descriptive datastore names for identification
- Tag datastores for cost allocation and organization

MAY:
- Automate datastore creation with CloudFormation or Terraform

## Safety Ratings

```
safety_ratings:
  - "list-datastores: GREEN — read-only enumeration of existing datastores"
  - "CreateDatastore: YELLOW — creates a new datastore resource with associated costs"
  - "kms:CreateGrant: RED — grants HealthImaging access to KMS key for encryption"
  - "service-quotas list-service-quotas: GREEN — read-only quota check"
```

## Escalation Conditions

- Datastore creation fails with internal service errors after IAM and KMS fixes
- KMS key policy changes require security team approval
- Service quota increase request is denied or delayed
- Multiple datastores needed across regions for compliance requirements

## Data Sensitivity

- `aws medical-imaging list-datastores`: LOW — returns datastore metadata only
- `aws kms describe-key`: LOW — returns key metadata, not key material
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions
- Datastore contents: HIGH — contains protected health information (PHI)

## Prohibited Actions

- NEVER create datastores without confirming encryption requirements (PHI data requires encryption)
- NEVER use AWS-managed keys when customer-managed keys are required by compliance
- NEVER delete existing datastores to work around quota limits
- NEVER share KMS key grants broader than necessary for HealthImaging

## Phase 3 — Rollback

1. If a datastore was created with wrong encryption: delete it before any data is imported: `aws medical-imaging delete-datastore --datastore-id <id>`
2. If KMS grants were created incorrectly: revoke the grant: `aws kms revoke-grant --key-id <key-id> --grant-id <grant-id>`
3. If quota increase was requested in error: contact AWS Support to cancel
4. Verify no PHI data was stored in incorrectly configured datastores
5. Document the rollback action and notify the compliance team

## Common Issues

- symptoms: "CreateDatastore returns AccessDeniedException"
  diagnosis: "IAM policy missing medical-imaging:CreateDatastore permission."
  resolution: "Add medical-imaging:CreateDatastore to the IAM policy. If using KMS, also add kms:CreateGrant and kms:DescribeKey."

- symptoms: "CreateDatastore fails with KMS key error"
  diagnosis: "KMS key policy does not allow HealthImaging service to use the key."
  resolution: "Update KMS key policy to allow medical-imaging.amazonaws.com principal with kms:CreateGrant and kms:DescribeKey."

## Output Format

```yaml
root_cause: "datastore_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Automate datastore provisioning via IaC"
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
