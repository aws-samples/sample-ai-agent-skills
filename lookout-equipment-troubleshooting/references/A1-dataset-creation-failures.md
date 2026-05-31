---
title: "A1 — Dataset Creation Failures"
description: "Diagnose Amazon Lookout for Equipment dataset creation failures"
status: active
severity: HIGH
triggers: ["cannot create dataset", "dataset creation failed", "CreateDataset error"]
owner: devops-agent
objective: "Resolve dataset creation issues and establish valid dataset for sensor data ingestion"
context: "Datasets define the schema for sensor data. Each dataset maps to one piece of equipment with multiple components and sensors."
---

## Phase 1 — Triage

MUST:
- List existing datasets: `aws lookoutequipment list-datasets`
- Check for duplicate dataset names: `aws lookoutequipment describe-dataset --dataset-name <name>`
- Verify IAM permissions for lookoutequipment:CreateDataset
- Validate the schema JSON structure matches required format

SHOULD:
- Check CloudTrail for CreateDataset errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDataset --max-results 10`
- Verify the S3 bucket exists and is in the same region
- Check service quotas for maximum datasets per account

MAY:
- Review the dataset schema file for JSON syntax errors
- Check if the region supports Lookout for Equipment

## Phase 2 — Remediate

MUST:
- Fix schema JSON to match required format: component name → sensor columns mapping
- Ensure dataset name is unique within the account and region
- Grant lookoutequipment:CreateDataset permission to the calling principal
- Verify S3 bucket is in the same region as the Lookout for Equipment resource

SHOULD:
- Use the console to validate schema before CLI creation
- Tag datasets for cost tracking and organization

MAY:
- Create datasets via CloudFormation for reproducibility
- Set up a naming convention for datasets across environments

## Common Issues

- symptoms: "CreateDataset returns ValidationException with schema error"
  diagnosis: "Schema JSON does not match required format. Components must have unique names and sensors must have unique column names."
  resolution: "Validate schema JSON structure. Each component needs a ComponentName and Columns array with Name and Type fields."

- symptoms: "CreateDataset returns ResourceAlreadyExistsException"
  diagnosis: "A dataset with the same name already exists in this account and region."
  resolution: "Use a different name or delete the existing dataset if it is no longer needed: aws lookoutequipment delete-dataset --dataset-name <name>."

- symptoms: "CreateDataset returns AccessDeniedException"
  diagnosis: "The calling principal lacks lookoutequipment:CreateDataset permission."
  resolution: "Attach a policy granting lookoutequipment:CreateDataset to the IAM user or role."

## Safety Ratings

```
safety_ratings:
  - "list-datasets: GREEN — read-only enumeration"
  - "describe-dataset: GREEN — read-only metadata retrieval"
  - "CreateDataset: YELLOW — creates a new resource, non-destructive but adds to account"
  - "delete-dataset: RED — permanently removes dataset and all associated models and schedulers"
```

## Escalation Conditions

- Dataset creation fails with internal service errors after schema and permission fixes
- Service quota limit reached and quota increase request is pending
- Schema validation passes locally but fails on the API with unclear error messages
- Dataset creation needed urgently for production predictive maintenance pipeline

## Data Sensitivity

- `aws lookoutequipment list-datasets`: LOW — returns dataset names and metadata only
- `aws lookoutequipment describe-dataset`: LOW — returns schema and configuration metadata
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions
- Dataset schema: LOW — contains sensor column names, not actual sensor values

## Prohibited Actions

- NEVER delete an existing dataset without confirming it has no active models or inference schedulers
- NEVER suggest creating datasets with placeholder schemas — schema must match actual sensor data
- NEVER modify a dataset schema after data has been ingested (schemas are immutable after ingestion)
- NEVER create datasets in unsupported regions without verifying regional availability first

## Phase 3 — Rollback

1. If a dataset was created with incorrect schema, delete it before any data ingestion: `aws lookoutequipment delete-dataset --dataset-name <name>`
2. If the wrong S3 role ARN was specified, the dataset must be deleted and recreated (role ARN is immutable)
3. Document the correct schema and store it in version control for future reference
4. Verify no dependent resources (models, schedulers) exist before deleting: `aws lookoutequipment list-models --dataset-name-begins-with <prefix>`
5. Notify the data engineering team of any dataset recreation to update pipeline configurations

## Output Format

```yaml
root_cause: "dataset_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details from API response>"
severity: HIGH
mitigation:
  immediate: "Fix schema or permissions and retry creation"
  long_term: "Implement schema validation in CI/CD pipeline"
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
