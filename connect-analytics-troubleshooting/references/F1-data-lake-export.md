---
title: "F1 — Data Lake Export"
description: "Diagnose Contact Lens data lake export issues"
status: active
severity: MEDIUM
triggers: ["data lake export failed", "no data in data lake", "export missing", "analytics export error"]
owner: devops-agent
objective: "Resolve data lake export configuration and data delivery issues"
context: "Contact Lens data can be exported to a data lake (S3) for analytics and reporting. Export has a processing delay of 15-60 minutes after contact completion. Data includes transcripts, sentiment, categories, and contact metadata."
---

## Phase 1 — Triage

MUST:
- Check data lake configuration in the Connect console
- Verify S3 bucket for data lake output: `aws s3 ls s3://<data-lake-bucket>/`
- Check if recent contacts have exported data (allow 15-60 minutes processing delay)

SHOULD:
- Verify S3 bucket permissions for the Connect service
- Check if the data lake export is enabled for the correct data types
- Verify the export is configured for the correct Connect instance

MAY:
- Check CloudWatch for data lake export errors
- Review CloudTrail for data lake configuration changes
- Compare expected vs actual data volume in the export bucket

## Phase 2 — Remediate

MUST:
- Enable data lake export in the Connect console if not configured
- Verify S3 bucket policy grants Connect service write permissions
- Allow sufficient time for processing (15-60 minutes after contact completion)

SHOULD:
- Configure export for all required data types (contacts, agents, queues, analytics)
- Set up S3 event notifications for new data delivery
- Verify data format and schema match downstream analytics requirements

MAY:
- Configure AWS Glue crawler for automatic schema discovery
- Set up Amazon Athena for SQL-based analytics on exported data
- Implement data partitioning for query performance

## Safety Ratings

```
safety_ratings:
  - "s3 ls: GREEN — read-only bucket listing"
  - "data lake configuration: YELLOW — modifies export settings"
  - "s3 bucket policy modification: RED — affects bucket access control"
  - "Glue crawler creation: YELLOW — creates data catalog resource"
```

## Escalation Conditions

- No data exported after 2 hours for contacts that have completed
- Data lake export produces incomplete or corrupted data
- Export volume significantly lower than expected contact volume
- S3 bucket storage costs exceed budget due to export volume

## Data Sensitivity

- `aws s3 ls` (data lake bucket): MEDIUM — reveals export file structure
- Exported contact data: HIGH — contains transcripts, sentiment, and contact metadata
- Data lake configuration: MEDIUM — reveals export settings and data types
- Athena query results: HIGH — may contain aggregated customer interaction data

## Prohibited Actions

- NEVER assume missing data means a configuration error if the contact ended less than 1 hour ago
- NEVER delete exported data without verifying retention requirements
- NEVER grant public access to the data lake S3 bucket
- NEVER query exported data without proper authorization

## Phase 3 — Rollback

1. Revert data lake configuration if changes caused export failures
2. Restore S3 bucket policy if changes affected other services
3. Delete Glue crawlers or Athena tables if created in error
4. Verify data export resumes for new contacts after changes
5. Document the export issue and resolution

## Common Issues

- symptoms: "Data lake bucket is empty despite active Contact Lens usage"
  diagnosis: "Data lake export not enabled in the Connect console — it is not enabled by default."
  resolution: "Enable data lake export in the Connect console under Analytics > Data lake."

## Output Format

```yaml
root_cause: "data_lake_export — <specific_cause>"
evidence:
  - type: export_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix data lake export configuration"
  long_term: "Implement export monitoring and data pipeline validation"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
