---
title: "C2 — Manual Step Issues"
description: "Diagnose Incident Manager manual step problems in runbooks and incident workflows"
status: active
severity: MEDIUM
triggers:
  - "manual step"
  - "manual approval"
  - "step not appearing"
  - "manual action"
  - "runbook manual"
owner: devops-agent
objective: "Resolve manual step issues in incident response workflows"
context: "SSM Automation documents can include manual approval steps that pause execution until a responder takes action. Common issues include steps not appearing in the incident timeline, approval timeouts, and notification failures for manual steps."
---

## Phase 1 — Triage

MUST:
- Check automation execution status: `aws ssm describe-automation-executions --filters Key=DocumentNamePrefix,Values=<doc-name>`
- Get execution details: `aws ssm get-automation-execution --automation-execution-id <id>`
- Check if execution is waiting on a manual step (status: Waiting)
- Verify the approver has been notified

SHOULD:
- Check the step timeout configuration
- Verify the approval notification was sent (SNS topic)
- Review the step's approval requirements

MAY:
- Approve manually: `aws ssm send-automation-signal --automation-execution-id <id> --signal-type Approve`
- Check CloudTrail for approval events

## Phase 2 — Remediate

MUST:
- If step timed out, restart the automation or approve manually
- If notification not sent, verify SNS topic configuration in the step
- If approver unknown, check the step's approvers list

SHOULD:
- Configure reasonable timeouts for manual steps
- Ensure multiple approvers are configured for redundancy

## Output Format

```yaml
root_cause: "manual_step — <specific_cause>"
evidence:
  - type: execution_status
    content: "<automation execution status>"
  - type: step_details
    content: "<manual step configuration>"
severity: MEDIUM
mitigation:
  immediate: "Approve pending step or fix notification"
  long_term: "Configure appropriate timeouts and multiple approvers"
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
