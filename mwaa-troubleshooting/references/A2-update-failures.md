---
title: "A2 — MWAA Environment Update Failures"
description: "Diagnose why an Amazon MWAA environment update fails"
status: active
severity: HIGH
triggers:
  - "environment update failed"
  - "UPDATE_FAILED"
  - "update stuck"
  - "environment not updating"
owner: devops-agent
objective: "Identify why an MWAA environment update fails and restore the environment to a healthy state"
context: "MWAA environment updates include configuration changes, requirements/plugins updates, and scaling changes. Updates take 20-30 minutes and the environment enters UPDATING status. Failures can leave the environment in an unhealthy state."
---

## Phase 1 — Triage

MUST:
- Check environment status: `aws mwaa get-environment --name <env-name> --query 'Environment.{Status:Status,LastUpdate:LastUpdate}'`
- Check last update details: `aws mwaa get-environment --name <env-name> --query 'Environment.LastUpdate'`
- Check CloudTrail for update events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateEnvironment --max-results 10`
- Verify requirements.txt if updated: `aws s3 cp s3://<bucket>/<requirements-path> -`
- Verify plugins.zip if updated: `aws s3 ls s3://<bucket>/<plugins-path>`

SHOULD:
- Check CloudWatch logs for startup errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-WebServer --filter-pattern "ERROR" --start-time <update-epoch>`
- Verify execution role permissions haven't changed: `aws iam list-attached-role-policies --role-name <execution-role-name>`
- Check if concurrent update was attempted: review CloudTrail for overlapping UpdateEnvironment calls

MAY:
- Compare current config with previous known-good config
- Check S3 object versions for requirements.txt and plugins.zip

## Phase 2 — Remediate

MUST:
- If requirements.txt caused failure, revert to previous version in S3 and trigger update
- If plugins.zip caused failure, revert to previous version and trigger update
- Wait for environment to stabilize before attempting another update

SHOULD:
- Test requirements.txt locally before applying to MWAA
- Validate plugins.zip structure matches Airflow plugin layout
- Avoid concurrent updates — wait for AVAILABLE status before updating

MAY:
- Create a new environment if the current one is unrecoverable
- Use S3 versioning to roll back to previous requirements/plugins

## Common Issues

- symptoms: "Environment stuck in UPDATING for over 60 minutes"
  diagnosis: "requirements.txt has a package that fails to install, blocking startup."
  resolution: "Revert requirements.txt to previous version in S3 and trigger a new update."

- symptoms: "UPDATE_FAILED after configuration change"
  diagnosis: "Invalid Airflow configuration override or incompatible setting."
  resolution: "Review the configuration overrides and remove or fix the invalid setting."

- symptoms: "Environment unhealthy after scaling change"
  diagnosis: "Max workers set too low for current workload or environment class insufficient."
  resolution: "Increase max workers or upgrade environment class."

## Output Format

```yaml
root_cause: "environment_update_failure — <specific_cause>"
evidence:
  - type: environment_status
    content: "<status and last update details>"
  - type: cloudtrail
    content: "<update event details>"
  - type: requirements_or_plugins
    content: "<file contents or error>"
severity: HIGH
mitigation:
  immediate: "Revert the change that caused the failure and wait for stabilization"
  long_term: "Test all changes in a staging environment before applying to production"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — updates operate on existing configurations
- Service Disruption Risk: HIGH — failed updates can leave environments in degraded state
- Compliance Impact: MEDIUM — configuration changes may affect audit posture

## Escalation Conditions
- Environment stuck in UPDATING state for more than 60 minutes
- Update causes environment to enter UPDATE_FAILED or UNAVAILABLE state
- Rollback to previous version is not possible
- Update involves Airflow version upgrade with breaking changes
- Multiple consecutive update failures on the same environment

## Data Sensitivity
- Environment configuration parameters — INTERNAL
- Airflow configuration overrides — INTERNAL
- Plugin and requirements file contents — INTERNAL
- Execution role modifications — CONFIDENTIAL
- Never log sensitive Airflow connection strings or variables

## Prohibited Actions
- Do NOT force-update an environment that is already in UPDATING state
- Do NOT downgrade Airflow versions (not supported by MWAA)
- Do NOT remove required IAM permissions during an update
- Do NOT change VPC configuration on an existing environment
- Do NOT update environments during peak DAG execution windows without approval
- Do NOT modify S3 DAG bucket path during update without migrating DAGs first

## Phase 3 — Rollback
- If update fails, check if environment auto-rolls back to previous configuration
- Restore previous requirements.txt and plugins.zip from S3 versioned objects
- Revert Airflow configuration overrides to last known working values
- If environment is stuck, contact AWS Support with environment ARN and timeline
- Document all configuration changes attempted and their outcomes
- Verify DAG execution resumes normally after rollback

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"
