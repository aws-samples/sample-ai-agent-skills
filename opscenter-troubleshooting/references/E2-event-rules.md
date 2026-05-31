---
title: "E2 — Event Rules"
description: "Diagnose EventBridge rule configuration for OpsItems"
status: active
severity: MEDIUM
triggers: ["event rule not matching", "wrong events", "rule pattern"]
owner: devops-agent
objective: "Resolve EventBridge rule configuration"
context: "EventBridge rules use event patterns to match events. Incorrect patterns cause missed or wrong OpsItem creation."
---

## Phase 1 — Triage

MUST:
- Get rule event pattern: `aws events describe-rule --name <rule> --query 'EventPattern'`
- Verify pattern matches expected event source and detail-type
- Check rule targets: `aws events list-targets-by-rule --rule <rule>`

SHOULD:
- Test pattern against sample events
- Verify input transformer maps fields correctly

MAY:
- Check EventBridge metrics for rule invocations

## Phase 2 — Remediate

MUST:
- Fix event pattern to match correct events
- Ensure target configuration creates valid OpsItems

SHOULD:
- Test with sample events before production use
- Document event patterns per OpsItem type

MAY:
- Use EventBridge schema registry for event discovery

## Common Issues

- symptoms: "Rule matches too many events — OpsItem flood"
  diagnosis: "Event pattern too broad."
  resolution: "Narrow event pattern with specific detail-type and source filters."

## Output Format

```yaml
root_cause: "event_rules — <specific_cause>"
evidence:
  - type: rule_pattern
    content: "<event pattern and matching behavior>"
severity: MEDIUM
mitigation:
  immediate: "Fix event pattern"
  long_term: "Test and validate event patterns"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — event rules define routing logic
- Service Disruption Risk: MEDIUM — incorrect rules can miss critical events
- Compliance Impact: MEDIUM — event rules affect incident detection coverage

## Escalation Conditions
- Event rules fail to match critical operational events
- Rule modifications cause false positive OpsItem creation
- Rule evaluation errors prevent event processing
- Multiple rules conflict causing duplicate or missed events
- Rule quota limits prevent adding new critical event rules

## Data Sensitivity
- Event rule patterns — INTERNAL
- Target configurations — INTERNAL
- Rule evaluation metrics — INTERNAL
- Event filtering criteria — INTERNAL
- Never expose event rule logic that reveals security monitoring patterns

## Prohibited Actions
- Do NOT delete event rules for critical operational events without replacement
- Do NOT create rules with overly broad patterns
- Do NOT modify production event rules without testing
- Do NOT bypass rule validation processes
- Do NOT ignore rule evaluation errors
- Do NOT create rules that route events to unauthorized targets

## Phase 3 — Rollback
- Restore deleted event rules from documented configurations
- Revert rule pattern changes to previous versions
- Re-enable disabled rules
- Verify event matching after rule changes
- Document rule modifications and their impact
- Update event rule management procedures

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
