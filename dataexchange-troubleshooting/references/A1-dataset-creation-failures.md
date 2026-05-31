---
title: "A1 — Dataset Creation Failures"
description: "Diagnose AWS Data Exchange dataset creation failures"
status: active
severity: HIGH
triggers: ["cannot create data set", "create-data-set error", "dataset creation failed", "AssetType invalid"]
owner: devops-agent
objective: "Resolve data set creation issues including permissions, asset types, and configuration"
context: "Data sets are the top-level container in Data Exchange. They have an asset type (S3_SNAPSHOT, API_GATEWAY_API, REDSHIFT_DATA_SHARE, LAKE_FORMATION_DATA_PERMISSION) that cannot be changed after creation."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for dataexchange:CreateDataSet: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names dataexchange:CreateDataSet`
- List existing data sets: `aws dataexchange list-data-sets`
- Review CloudTrail for CreateDataSet errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDataSet --max-results 10`

SHOULD:
- Verify the asset type is valid for the intended data source
- Check service quotas for maximum data sets per account

MAY:
- Review IAM policy conditions that may restrict data set creation
- Check for SCP restrictions in AWS Organizations

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for dataexchange:CreateDataSet
- Use correct asset type matching the data source (S3_SNAPSHOT, API_GATEWAY_API, etc.)
- Ensure data set name and description meet API requirements

SHOULD:
- Request service quota increase if limit reached
- Verify the account is registered as a Data Exchange provider

MAY:
- Use tags for data set organization and cost tracking
- Set up CloudFormation for repeatable data set creation

## Safety Ratings

```
safety_ratings:
  - "dataexchange list-data-sets: GREEN — read-only enumeration"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "dataexchange create-data-set: YELLOW — creates a new data set resource"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Data set creation fails with internal service errors after IAM fixes
- Service quota limit reached and increase request is pending
- SCP restrictions prevent data set creation and require organization admin
- Asset type requirements are unclear for the intended data source

## Data Sensitivity

- `aws dataexchange list-data-sets`: LOW — returns data set metadata
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- `data set configuration`: LOW — contains metadata, not actual data

## Prohibited Actions

- NEVER create data sets with incorrect asset types — asset type cannot be changed after creation
- NEVER delete existing data sets with active subscribers to fix creation issues
- NEVER share provider IAM credentials to debug subscriber-side creation issues
- NEVER create data sets in the wrong region without verifying subscriber expectations

## Phase 3 — Rollback

1. If a data set was created with wrong asset type, delete it (only if no revisions/subscribers): `aws dataexchange delete-data-set --data-set-id <id>`
2. If IAM permissions were modified, revert to previous policy version
3. Verify no orphaned data sets exist from failed creation attempts
4. Document the creation issue and correct configuration for future reference
5. Confirm the data set is accessible to intended subscribers after recreation

## Common Issues

- symptoms: "CreateDataSet returns AccessDeniedException"
  diagnosis: "IAM role lacks dataexchange:CreateDataSet permission."
  resolution: "Add dataexchange:CreateDataSet to the IAM policy."

## Output Format

```yaml
root_cause: "dataset_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Automate data set creation via IaC"
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
