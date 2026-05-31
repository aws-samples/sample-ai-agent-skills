---
title: "A1 — Matching Workflow Failures"
description: "Diagnose Entity Resolution matching workflow execution failures including permissions, configuration, and data errors"
status: active
severity: HIGH
triggers: ["matching workflow failed", "matching job error", "workflow execution failed", "StartMatchingJob error"]
owner: devops-agent
objective: "Resolve matching workflow execution failures"
context: "Matching workflows process input data to find matching records. They support RULE_MATCHING, ML_MATCHING, and PROVIDER resolution types. Failures can occur at job start, during processing, or at output writing."
---

## Phase 1 — Triage

MUST:
- Get workflow details: `aws entityresolution get-matching-workflow --workflow-name <workflow-name>`
- Check recent job status: `aws entityresolution get-matching-job --workflow-name <workflow-name> --job-id <job-id>`
- Verify the service role permissions: `aws iam get-role --role-name <role-name>`

SHOULD:
- Check input data source (Glue table): `aws glue get-table --database-name <db> --name <table>`
- Verify S3 output location permissions
- Review CloudTrail for StartMatchingJob events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StartMatchingJob --max-results 10`

MAY:
- Check CloudWatch metrics for Entity Resolution
- Verify KMS key permissions if encryption is configured
- Review input data sample for format issues

## Phase 2 — Remediate

MUST:
- Fix service role permissions for Glue, S3, and KMS access
- Correct workflow configuration (input sources, output location, resolution type)
- Ensure input Glue tables exist and are accessible

SHOULD:
- Verify schema mappings match Glue table columns
- Test with a small dataset before running on full data
- Check S3 bucket policy allows the service role to write output

MAY:
- Set up CloudWatch alarms for job failures
- Implement job monitoring and notification

## Safety Ratings

```
safety_ratings:
  - "get-matching-workflow: GREEN — read-only workflow configuration"
  - "get-matching-job: GREEN — read-only job status"
  - "start-matching-job: YELLOW — initiates data processing job"
  - "delete-matching-workflow: RED — permanently removes workflow configuration"
```

## Escalation Conditions

- Matching job fails with internal service errors after fixing permissions and configuration
- Job processes data but produces no output or empty results
- Large dataset job fails after hours of processing
- Service role permissions cannot be determined from error messages

## Data Sensitivity

- `aws entityresolution get-matching-workflow`: LOW — returns workflow configuration
- `aws entityresolution get-matching-job`: LOW — returns job metadata and status
- `aws glue get-table`: LOW — returns table schema metadata
- Input/output data: HIGH — may contain PII or sensitive business data

## Prohibited Actions

- NEVER delete a matching workflow without verifying no downstream processes depend on it
- NEVER modify input Glue tables while a matching job is running
- NEVER grant the service role access to unrelated S3 buckets or Glue databases
- NEVER run matching jobs on production data without testing on a sample first

## Phase 3 — Rollback

1. If a job was started in error, it cannot be cancelled — wait for completion or failure
2. If output data was written incorrectly, delete the output files from S3
3. If the workflow was misconfigured, update with correct settings
4. If the service role was modified, restore previous permissions
5. Document the failure and resolution for future reference

## Common Issues

- symptoms: "StartMatchingJob fails with AccessDeniedException"
  diagnosis: "The service role lacks permissions for Glue, S3, or KMS resources."
  resolution: "Grant the service role glue:GetTable, glue:GetDatabase, s3:GetObject, s3:PutObject, and kms:Decrypt permissions."

- symptoms: "Job completes but output is empty"
  diagnosis: "No records matched based on the current rules or ML configuration, or schema mapping is incorrect."
  resolution: "Verify schema mappings. Check matching rules or ML configuration. Test with known matching records."

- symptoms: "Job fails with ResourceNotFoundException"
  diagnosis: "The input Glue table, database, or schema mapping does not exist."
  resolution: "Verify all referenced resources exist. Check for typos in resource names."

## Output Format

```yaml
root_cause: "matching_workflow — <specific_cause>"
evidence:
  - type: workflow_error
    content: "<job failure details>"
severity: HIGH
mitigation:
  immediate: "Fix workflow configuration or permissions"
  long_term: "Implement pre-job validation and monitoring"
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
