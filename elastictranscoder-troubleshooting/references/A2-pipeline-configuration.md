---
title: "A2 — Pipeline Configuration"
description: "Diagnose Elastic Transcoder pipeline configuration issues"
status: active
severity: MEDIUM
triggers: ["pipeline misconfigured", "pipeline settings wrong", "update pipeline failed"]
owner: devops-agent
objective: "Resolve pipeline configuration issues including status, notifications, and bucket settings"
context: "Pipelines can be Active or Paused. Configuration includes input/output buckets, IAM role, and SNS topics."
---

## Phase 1 — Triage

MUST:
- Read pipeline configuration: `aws elastictranscoder read-pipeline --id <pipeline-id>`
- Verify pipeline status is Active: check the Status field in pipeline details
- Verify IAM role is still valid: `aws iam get-role --role-name <role-name>`

SHOULD:
- Check SNS topic configuration if notifications are expected
- Verify S3 bucket policies have not changed since pipeline creation
- Check if pipeline was recently paused or modified

MAY:
- Review CloudTrail for UpdatePipeline events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdatePipeline --max-results 10`

## Phase 2 — Remediate

MUST:
- Reactivate paused pipelines if they should be active: `aws elastictranscoder update-pipeline-status --id <pipeline-id> --status Active`
- Fix IAM role if permissions were removed
- Update bucket references if buckets were moved or renamed

SHOULD:
- Configure SNS notifications for all job states (Completed, Warning, Error)
- Verify output key prefix is correct

MAY:
- Set up CloudWatch alarms for pipeline job failures

## Safety Ratings

```
safety_ratings:
  - "read-pipeline: GREEN — read-only pipeline inspection"
  - "update-pipeline-status: YELLOW — changes pipeline processing state"
  - "update-pipeline: YELLOW — modifies pipeline configuration"
  - "iam get-role: GREEN — read-only role inspection"
```

## Escalation Conditions

- Pipeline status cannot be changed from Paused to Active
- Pipeline configuration updates fail with internal service errors
- IAM role associated with the pipeline has been deleted and cannot be recreated
- Pipeline is processing jobs incorrectly despite correct configuration

## Data Sensitivity

- `aws elastictranscoder read-pipeline`: LOW — returns pipeline configuration
- `aws iam get-role`: LOW — returns role metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER pause a production pipeline without explicit approval
- NEVER change the IAM role on a pipeline with active jobs
- NEVER modify output bucket or key prefix on a pipeline with active jobs
- NEVER delete and recreate a pipeline as a troubleshooting step without approval

## Phase 3 — Rollback

1. If pipeline status was changed, revert it: `aws elastictranscoder update-pipeline-status --id <pipeline-id> --status <original-status>`
2. If pipeline configuration was modified, restore original settings using the read-pipeline output captured before changes
3. If SNS topics were changed, restore original topic ARNs
4. Verify all active jobs are still processing correctly after any configuration changes
5. Document all configuration changes made during troubleshooting

## Common Issues

- symptoms: "Jobs submitted but not processing"
  diagnosis: "Pipeline status is Paused."
  resolution: "Reactivate pipeline: aws elastictranscoder update-pipeline-status --id <id> --status Active."

- symptoms: "Pipeline update returns ValidationException"
  diagnosis: "Referenced S3 bucket or IAM role no longer exists."
  resolution: "Recreate the missing resource or update pipeline to reference existing resources."

## Output Format

```yaml
root_cause: "pipeline_configuration — <specific_cause>"
evidence:
  - type: config_issue
    content: "<configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix pipeline configuration"
  long_term: "Monitor pipeline status with CloudWatch"
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
