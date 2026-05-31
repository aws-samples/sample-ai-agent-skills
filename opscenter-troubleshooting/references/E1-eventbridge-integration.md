---
title: "E1 — EventBridge Integration"
description: "Diagnose EventBridge to OpsCenter integration issues"
status: active
severity: HIGH
triggers: ["EventBridge not creating OpsItems", "integration broken", "events not reaching OpsCenter"]
owner: devops-agent
objective: "Resolve EventBridge to OpsCenter integration"
context: "EventBridge rules can create OpsItems automatically. Rules must target SSM OpsCenter with proper input transformation."
---

## Phase 1 — Triage

MUST:
- List EventBridge rules targeting OpsCenter: `aws events list-rules --query 'Rules[?contains(Name, `SSM`) || contains(Name, `OpsItem`)]'`
- Check rule state (ENABLED/DISABLED): `aws events describe-rule --name <rule>`
- Verify rule target is SSM OpsCenter

SHOULD:
- Check rule event pattern matches expected events
- Verify IAM role for EventBridge to SSM

MAY:
- Test rule with manual event: `aws events put-events`

## Phase 2 — Remediate

MUST:
- Enable disabled rules
- Fix event pattern to match expected events
- Ensure IAM role allows events:PutEvents and ssm:CreateOpsItem

SHOULD:
- Configure input transformer for proper OpsItem fields
- Include dedup string in input transformer

MAY:
- Set up dead-letter queue for failed deliveries

## Common Issues

- symptoms: "CloudWatch alarms fire but no OpsItems created"
  diagnosis: "EventBridge rule for CloudWatch → OpsCenter is disabled or missing."
  resolution: "Enable or create EventBridge rule with CloudWatch alarm event pattern targeting OpsCenter."

## Output Format

```yaml
root_cause: "eventbridge_integration — <specific_cause>"
evidence:
  - type: rule_config
    content: "<EventBridge rule state and configuration>"
severity: HIGH
mitigation:
  immediate: "Fix EventBridge rule"
  long_term: "Monitor EventBridge rule delivery metrics"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — EventBridge integration is event routing
- Service Disruption Risk: MEDIUM — broken integration prevents automated OpsItem creation
- Compliance Impact: MEDIUM — missed events can affect incident response SLAs

## Escalation Conditions
- EventBridge rules stop triggering OpsItem creation
- Event pattern changes cause critical events to be missed
- EventBridge permissions prevent OpsCenter integration
- Event delivery failures cause operational blind spots
- Cross-account event routing fails

## Data Sensitivity
- Event patterns and rules — INTERNAL
- Event payloads — INTERNAL (may contain CONFIDENTIAL data)
- EventBridge IAM configurations — INTERNAL
- Cross-account event bus configurations — INTERNAL
- Never expose event payloads containing sensitive data

## Prohibited Actions
- Do NOT disable EventBridge rules for critical operational events
- Do NOT create overly broad event patterns that generate noise
- Do NOT ignore EventBridge delivery failures
- Do NOT modify event patterns without testing
- Do NOT bypass EventBridge permissions for troubleshooting
- Do NOT route sensitive events to unauthorized event buses

## Phase 3 — Rollback
- Restore previous EventBridge rule configurations
- Re-enable disabled event rules
- Revert event pattern changes
- Restore cross-account event routing if broken
- Verify event delivery after configuration changes
- Document EventBridge integration changes and their impact

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
