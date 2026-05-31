---
title: "A2 — Incident Creation Issues"
description: "Diagnose Incident Manager incident creation failures including triggers and permissions"
status: active
severity: HIGH
triggers:
  - "incident not created"
  - "start incident failed"
  - "incident creation error"
  - "alarm not creating incident"
  - "incident trigger"
owner: devops-agent
objective: "Resolve incident creation failures and ensure incidents are properly triggered"
context: "Incidents can be created manually, by CloudWatch alarms, or by EventBridge rules. Common issues include wrong response plan ARN, missing permissions, alarm misconfiguration, and replication set issues."
---

## Phase 1 — Triage

MUST:
- Verify response plan exists: `aws ssm-incidents get-response-plan --arn <plan-arn>`
- Check replication set status: `aws ssm-incidents get-replication-set`
- If alarm-triggered, verify alarm action is the response plan ARN: `aws cloudwatch describe-alarms --alarm-names <alarm>`
- Check IAM permissions for `ssm-incidents:StartIncident`
- Check CloudTrail for StartIncident events

SHOULD:
- Verify the alarm is in ALARM state
- Check if the response plan has valid engagements configured
- Verify the service-linked role exists for alarm integration

MAY:
- Test manual incident creation: `aws ssm-incidents start-incident --response-plan-arn <arn> --title "Test Incident"`
- Check EventBridge rules if event-triggered

## Phase 2 — Remediate

MUST:
- If alarm action wrong, update to response plan ARN: `aws cloudwatch put-metric-alarm --alarm-name <name> --alarm-actions <response-plan-arn>`
- If permissions missing, add `ssm-incidents:StartIncident` to the triggering principal
- If replication set issue, fix replication set configuration

SHOULD:
- Test incident creation after fixing configuration
- Verify engagements fire correctly after incident creation

## Output Format

```yaml
root_cause: "incident_creation — <specific_cause>"
evidence:
  - type: response_plan
    content: "<response plan details>"
  - type: alarm_config
    content: "<alarm action configuration>"
severity: HIGH
mitigation:
  immediate: "Fix trigger configuration or permissions"
  long_term: "Test incident creation regularly with simulated alarms"
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
