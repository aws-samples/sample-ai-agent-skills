---
title: "E1 — CloudWatch Alarm Integration"
description: "Diagnose Incident Manager integration issues with CloudWatch alarms"
status: active
severity: HIGH
triggers:
  - "cloudwatch alarm incident"
  - "alarm not triggering incident"
  - "alarm integration"
  - "alarm action response plan"
owner: devops-agent
objective: "Resolve CloudWatch alarm integration with Incident Manager"
context: "CloudWatch alarms can automatically create incidents by using a response plan ARN as the alarm action. Common issues include wrong alarm action, missing permissions, alarm not in ALARM state, and response plan ARN format errors."
---

## Phase 1 — Triage

MUST:
- Check alarm configuration: `aws cloudwatch describe-alarms --alarm-names <alarm>`
- Verify alarm action is the response plan ARN (not SNS topic)
- Check alarm state: should be ALARM to trigger
- Verify response plan exists: `aws ssm-incidents get-response-plan --arn <plan-arn>`
- Check CloudTrail for StartIncident events from the alarm

SHOULD:
- Verify the alarm has the correct metric and threshold
- Check if the alarm has been in ALARM state recently: `aws cloudwatch describe-alarm-history --alarm-name <alarm>`
- Verify service-linked role for alarm integration

MAY:
- Test by manually setting alarm state: `aws cloudwatch set-alarm-state --alarm-name <alarm> --state-value ALARM --state-reason "Testing"`
- Check for alarm action suppression

## Phase 2 — Remediate

MUST:
- Set alarm action to response plan ARN: `aws cloudwatch put-metric-alarm --alarm-name <alarm> --alarm-actions <response-plan-arn>`
- Verify the response plan ARN format is correct
- Ensure the alarm metric and threshold are correctly configured

SHOULD:
- Test alarm-to-incident flow after configuration
- Configure OK actions to update incident status

## Output Format

```yaml
root_cause: "cloudwatch_alarm — <specific_cause>"
evidence:
  - type: alarm_config
    content: "<alarm configuration>"
  - type: alarm_history
    content: "<alarm state history>"
severity: HIGH
mitigation:
  immediate: "Fix alarm action to reference response plan ARN"
  long_term: "Manage alarm-response plan mappings via IaC"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete response plans that are actively referenced by CloudWatch alarms
- Never modify escalation plans during active incidents
- Never remove contact channels without confirming alternative notification paths exist
- Never delete incident records or post-incident analysis data
- Never disable automation runbooks tied to active response plans without backup procedures

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous response plan configuration if incident detection breaks
- Revert escalation plan changes if notifications fail to reach responders
- Re-enable previous contact channel settings if engagement plans stop working

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
