---
title: "A1 — Schedule Creation Failures"
description: "Diagnose EventBridge Scheduler schedule creation failures"
status: active
severity: HIGH
triggers:
  - "schedule creation failed"
  - "cannot create schedule"
  - "scheduler error"
  - "schedule validation"
owner: devops-agent
objective: "Identify the root cause of EventBridge Scheduler creation failures"
context: "Schedule creation can fail due to invalid expressions, IAM role issues, target configuration errors, schedule group limits, or invalid parameters. Scheduler uses a separate API from EventBridge rules."
---

## Phase 1 — Triage

MUST:
- Check if the schedule exists: `aws scheduler get-schedule --name <schedule-name>`
- Verify the IAM execution role: `aws iam get-role --role-name <role-name>`
- Check the schedule expression syntax (cron, rate, or at)
- Verify the target configuration (ARN, role, input)

SHOULD:
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=scheduler.amazonaws.com --max-results 20`
- Verify the schedule group exists: `aws scheduler list-schedule-groups`
- Check service quotas for schedule count
- Verify the target service is supported

MAY:
- Check for conflicting schedule names in the group
- Verify KMS key if using customer-managed encryption
- Review schedule group tags and configuration

## Phase 2 — Remediate

MUST:
- Fix the schedule expression syntax
- Ensure IAM role trusts scheduler.amazonaws.com and has target permissions
- Verify target ARN is correct and accessible

SHOULD:
- Use schedule groups to organize related schedules
- Configure retry policy and DLQ for failed invocations
- Test the schedule with a short interval before production deployment

MAY:
- Set up CloudWatch alarms for schedule failures
- Use IaC for consistent schedule deployments

## Common Issues

- symptoms: "ValidationException on schedule creation"
  diagnosis: "Schedule expression, target ARN, or role ARN is invalid."
  resolution: "Verify cron/rate/at expression syntax. Check target ARN format. Verify role exists."

- symptoms: "Schedule created but target not invoked"
  diagnosis: "IAM execution role lacks permission to invoke the target."
  resolution: "Add target-specific permissions to the execution role (e.g., lambda:InvokeFunction, sqs:SendMessage)."

- symptoms: "Schedule group quota exceeded"
  diagnosis: "Maximum number of schedules in the group or account reached."
  resolution: "Delete unused schedules or request a quota increase."

## Output Format

```yaml
root_cause: "schedule_creation — <specific_cause>"
evidence:
  - type: schedule
    content: "<schedule configuration or error>"
  - type: iam_role
    content: "<execution role and permissions>"
  - type: cloudtrail
    content: "<creation error events>"
severity: HIGH
mitigation:
  immediate: "Fix expression, role, or target configuration"
  long_term: "Standardize schedule creation with IaC templates"
```


## Safety Ratings
```
safety_ratings:
  - "Check schedule status: GREEN — read-only API call"
  - "Verify IAM execution role: GREEN — read-only IAM inspection"
  - "Check CloudTrail for errors: GREEN — read-only audit log query"
  - "Fix schedule expression syntax: GREEN — corrects configuration without side effects"
  - "Update IAM execution role permissions: YELLOW — changes access scope for the schedule"
  - "Delete and recreate schedule: YELLOW — brief gap in scheduled invocations"
```

## Escalation Conditions
- Schedule drives production workflow automation
- Schedule creation failure blocks critical business process
- IAM role changes affect multiple schedules or services
- Schedule group quota reached requiring architecture review
- KMS key access issues affecting encrypted schedules

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Schedule target configuration: downstream service identifiers"
    - "IAM execution role ARN: permission boundary identifier"
    - "Schedule input payload: may contain business data"
  handling: "Do not expose schedule input payloads or target ARNs externally."
```

## Prohibited Actions
- NEVER delete a schedule group without confirming all schedules within it are no longer needed
- NEVER use overly permissive IAM roles (e.g., AdministratorAccess) for schedule execution
- NEVER create schedules with sub-minute rates in production without understanding cost implications
- NEVER modify a production schedule without testing the expression in a non-production environment first

## Phase 3 — Rollback
- If schedule expression was changed: update the schedule with the previous expression
- If IAM execution role was modified: revert IAM policy to previous version
- If schedule was deleted and recreated: verify the new schedule fires at the expected time
- If schedule group was created: delete the group if not needed (must be empty first)
- If KMS key policy was changed: restore previous key policy from CloudTrail history

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
