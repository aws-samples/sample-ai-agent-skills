---
title: "G1 — Data Loading"
description: "Diagnose Neptune data loading and migration issues"
status: active
severity: MEDIUM
triggers:
  - "data loading"
  - "data migration"
  - "import data"
  - "load csv"
owner: devops-agent
objective: "Identify and resolve Neptune data loading issues"
context: "Neptune supports bulk loading from S3 (recommended for large datasets) and individual query inserts. The bulk loader requires specific data formats and an IAM role with S3 access."
---

## Phase 1 — Triage

MUST:
- Check bulk loader status: `curl https://<endpoint>:8182/loader`
- Check specific load job: `curl 'https://<endpoint>:8182/loader/<load-id>?details=true&errors=true'`
- Verify IAM role: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].AssociatedRoles'`
- Verify S3 data accessibility: `aws s3 ls s3://<bucket>/<prefix>/`

SHOULD:
- Validate data format before loading
- Check S3 bucket region matches Neptune region
- Verify data file encoding (UTF-8 required)
- Check for duplicate IDs in source data

MAY:
- Test with small data sample first
- Monitor cluster metrics during loading
- Check Neptune audit logs for load errors

## Phase 2 — Remediate

MUST:
- Fix data format issues identified in loader errors
- Ensure IAM role has correct S3 permissions
- Retry loading after fixing issues

SHOULD:
- Use bulk loader for datasets > 1000 records
- Set appropriate parallelDegree for loading speed
- Use updateSingleCardinalityProperties for idempotent loads

MAY:
- Implement data validation pipeline before loading
- Set up automated data loading workflows

## Common Issues

- symptoms: "Loader reports PARSING_ERROR"
  diagnosis: "Data file format does not match expected schema."
  resolution: "For Gremlin CSV: verify headers (~id, ~label, ~from, ~to). For RDF: validate syntax."

- symptoms: "Loader hangs with no progress"
  diagnosis: "Large file with single-threaded loading or network issue to S3."
  resolution: "Increase parallelDegree. Verify VPC endpoint or NAT gateway for S3 access."

- symptoms: "Duplicate vertex/edge errors"
  diagnosis: "Source data contains duplicate IDs."
  resolution: "Deduplicate source data or use updateSingleCardinalityProperties mode."

## Output Format

```yaml
root_cause: "data_loading — <specific_cause>"
evidence:
  - type: loader_status
    content: "<loader job details>"
  - type: data_format
    content: "<data format validation>"
severity: MEDIUM
mitigation:
  immediate: "Fix data issues and retry loading"
  long_term: "Implement data validation and automated loading pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — data loading involves potentially sensitive graph data
- Service Disruption Risk: MEDIUM — loading operations consume cluster resources
- Compliance Impact: MEDIUM — data loading must comply with data governance policies

## Escalation Conditions
- Data loading fails with persistent format or validation errors
- Loading operations cause cluster performance degradation
- Loaded data contains unexpected or incorrect values
- Loading IAM role permissions are insufficient
- Data loading from S3 fails due to access or network issues

## Data Sensitivity
- Source data files — CONFIDENTIAL
- S3 bucket paths — INTERNAL
- IAM role configurations — INTERNAL
- Data format specifications — INTERNAL
- Never expose source data samples in error logs

## Prohibited Actions
- Do NOT load data from untrusted or unvalidated sources
- Do NOT skip data format validation before loading
- Do NOT run large data loads during peak query hours without approval
- Do NOT grant data loading roles access to unrelated S3 resources
- Do NOT ignore data loading error reports
- Do NOT load data without a rollback plan

## Phase 3 — Rollback
- Use Neptune's drop graph or delete operations to remove incorrectly loaded data
- Restore from snapshot if bulk removal is not feasible
- Revert IAM role permissions added for data loading
- Verify data integrity after rollback
- Document data loading issues and format requirements
- Update data validation procedures based on findings

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
