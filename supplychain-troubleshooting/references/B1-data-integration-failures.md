---
title: "B1 — Data Integration Failures"
description: "Diagnose data integration flow failures"
status: active
severity: HIGH
triggers: ["data integration failed", "flow execution error", "data ingestion failure"]
owner: devops-agent
objective: "Resolve data integration flow failures in Supply Chain"
context: "Data integration flows connect external sources to the Supply Chain data lake. Flows can fail silently with partial data."
---

## Phase 1 — Triage

MUST:
- List data integration flows: `aws scn list-data-integration-flows --instance-id <instance-id>`
- Get flow details: `aws scn get-data-integration-flow --instance-id <instance-id> --name <flow-name>`
- Verify S3 source data exists and is accessible: `aws s3 ls s3://<source-bucket>/<prefix>/`

SHOULD:
- Check IAM role permissions for the data integration flow
- Verify source data format matches expected schema

MAY:
- Review CloudTrail for flow execution events
- Check CloudWatch metrics for flow execution duration and errors

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions: s3:GetObject on source, scn:* on instance
- Fix source data format to match expected schema
- Ensure S3 bucket and instance are in the same region

SHOULD:
- Implement data validation before ingestion
- Set up flow execution monitoring

MAY:
- Automate flow execution with scheduled triggers
- Implement error notification via SNS

## Safety Ratings

```
safety_ratings:
  - "list-data-integration-flows: GREEN — read-only flow enumeration"
  - "get-data-integration-flow: GREEN — read-only flow details"
  - "create-data-integration-flow: YELLOW — creates new data pipeline"
  - "s3 ls: GREEN — read-only bucket listing"
```

## Escalation Conditions

- Data integration flows fail consistently with internal service errors
- Large-scale data ingestion exceeds service quotas
- Source data format changes require flow reconfiguration
- Data quality issues in source affect downstream planning

## Data Sensitivity

- `aws scn list-data-integration-flows`: LOW — returns flow metadata
- S3 source data: HIGH — may contain proprietary supply chain data
- Flow execution logs: MEDIUM — may contain data references
- Data lake contents: HIGH — contains business-critical supply chain data

## Prohibited Actions

- NEVER delete data integration flows without confirming downstream impact
- NEVER modify source data format without coordinating with data providers
- NEVER use overly permissive IAM roles for data integration
- NEVER ignore partial data ingestion — always verify completeness

## Phase 3 — Rollback

1. If a flow was created incorrectly: delete it: `aws scn delete-data-integration-flow --instance-id <id> --name <flow-name>`
2. If IAM role permissions were broadened: revert to least-privilege
3. If source data was modified: restore from backup
4. Verify data lake datasets are consistent after flow changes
5. Document the failure and resolution for operations team

## Common Issues

- symptoms: "Data integration flow shows FAILED status"
  diagnosis: "IAM role lacks S3 read permissions or source data format is incorrect."
  resolution: "Fix IAM role permissions. Validate source data format matches expected schema."

- symptoms: "Flow completes but data lake dataset is empty"
  diagnosis: "Source data file format or folder structure doesn't match flow configuration."
  resolution: "Verify S3 file path, format (CSV/JSON/Parquet), and column names match flow expectations."

## Output Format

```yaml
root_cause: "data_integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix integration issue and retry"
  long_term: "Implement data validation pipeline"
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
