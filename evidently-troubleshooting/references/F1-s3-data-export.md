---
title: "F1 — S3 Data Export"
description: "Diagnose S3 data export failures for Evidently evaluation and event data"
status: active
severity: MEDIUM
triggers: ["S3 export missing", "no data in S3", "data delivery failed", "export error"]
owner: devops-agent
objective: "Resolve S3 data export issues for Evidently evaluation and event data"
context: "Data delivery to S3 is configured at the project level. It exports evaluation events and custom events for analysis."
---

## Phase 1 — Triage

MUST:
- Check project data delivery configuration: `aws evidently get-project --project <name> --query 'project.dataDelivery'`
- Verify S3 bucket exists: `aws s3api head-bucket --bucket <bucket-name>`
- Check S3 bucket policy allows Evidently to write: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Verify the S3 bucket is in the same region as the project
- Check for recent objects in the expected prefix: `aws s3 ls s3://<bucket>/<prefix>/ --recursive --max-items 10`
- Verify the project has active features generating evaluation events

MAY:
- Check CloudTrail for PutObject failures from the Evidently service
- Review S3 bucket encryption settings

## Phase 2 — Remediate

MUST:
- Configure data delivery if not set: `aws evidently update-project-data-delivery --project <name> --s3-destination '{"bucket":"<bucket>","prefix":"<prefix>"}'`
- Fix S3 bucket policy to allow Evidently service principal to write objects
- Ensure the bucket exists and is accessible

SHOULD:
- Set appropriate bucket prefix for data organization
- Configure S3 lifecycle rules for data retention management

MAY:
- Set up S3 event notifications for data processing pipelines
- Configure Athena tables for querying exported data

## Common Issues

- symptoms: "Data delivery configured but no objects in S3"
  diagnosis: "S3 bucket policy may not allow Evidently to write, or no evaluation events are being generated."
  resolution: "Verify: 1) Bucket policy allows evidently.amazonaws.com to PutObject, 2) Features are being evaluated (generating events), 3) Bucket name and prefix are correct."

## Safety Ratings

```
safety_ratings:
  - "get-project: GREEN — read-only project details"
  - "s3api head-bucket: GREEN — read-only bucket existence check"
  - "s3api get-bucket-policy: GREEN — read-only policy retrieval"
  - "update-project-data-delivery: YELLOW — modifies data export configuration"
```

## Escalation Conditions

- S3 bucket policy correct but data still not delivered after 1 hour
- Data export causing S3 bucket to exceed storage budget
- Cross-account S3 delivery fails despite correct bucket policy
- Exported data format does not match expected schema

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals bucket access permissions
- `aws evidently get-project`: LOW — returns project configuration
- Evidently data in S3: MEDIUM — contains evaluation events and entity identifiers

## Prohibited Actions

- NEVER modify S3 bucket policies on production buckets without reviewing existing permissions
- NEVER delete existing S3 objects to troubleshoot delivery issues
- NEVER disable S3 bucket encryption to work around delivery failures
- NEVER grant overly broad S3 permissions to fix delivery issues

## Phase 3 — Rollback

1. If data delivery was enabled in error, disable it: `aws evidently update-project-data-delivery --project <name>` with empty S3 destination
2. If S3 bucket policy was modified, revert to previous policy
3. If S3 lifecycle rules were changed, restore original rules
4. Verify no sensitive data was exposed through bucket policy changes
5. Document all S3 and project configuration changes

## Output Format

```yaml
root_cause: "s3_data_export — <specific_cause>"
evidence:
  - type: export_error
    content: "<S3 and data delivery configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 data delivery configuration"
  long_term: "Automate S3 bucket and delivery setup via IaC"
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
