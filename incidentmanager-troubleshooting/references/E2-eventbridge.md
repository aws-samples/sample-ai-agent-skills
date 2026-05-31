---
title: "E2 — EventBridge Integration"
description: "Diagnose Incident Manager integration issues with Amazon EventBridge"
status: active
severity: MEDIUM
triggers:
  - "eventbridge incident"
  - "event rule incident"
  - "eventbridge integration"
  - "event not creating incident"
owner: devops-agent
objective: "Resolve EventBridge integration with Incident Manager"
context: "EventBridge rules can trigger incident creation via Incident Manager response plans. Common issues include wrong target configuration, event pattern mismatches, missing IAM permissions, and rule not being enabled."
---

## Phase 1 — Triage

MUST:
- Check EventBridge rule: `aws events describe-rule --name <rule-name>`
- List rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify target is the response plan ARN with correct input transformer
- Check rule is enabled (State: ENABLED)
- Verify IAM role for EventBridge target has `ssm-incidents:StartIncident`

SHOULD:
- Check event pattern matches expected events
- Verify events are being generated: `aws events list-rules --name-prefix <prefix>`
- Check CloudTrail for EventBridge invocation events

MAY:
- Test with a put-events call: `aws events put-events --entries '[{"Source":"test","DetailType":"test","Detail":"{}"}]'`
- Check dead-letter queue for failed invocations

## Phase 2 — Remediate

MUST:
- Fix target configuration to reference response plan ARN
- Grant `ssm-incidents:StartIncident` to the EventBridge execution role
- Enable the rule if disabled
- Fix event pattern to match expected events

SHOULD:
- Configure input transformer to map event data to incident fields
- Set up dead-letter queue for failed invocations

## Output Format

```yaml
root_cause: "eventbridge — <specific_cause>"
evidence:
  - type: rule_config
    content: "<EventBridge rule configuration>"
  - type: target_config
    content: "<rule target configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix EventBridge rule target or permissions"
  long_term: "Manage EventBridge-Incident Manager integration via IaC"
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
