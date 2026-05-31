---
title: "C2 — Stack Configuration Issues"
description: "Diagnose CloudFormation stack-based coverage problems"
status: active
severity: MEDIUM
triggers: ["stack not monitored", "stack coverage error", "stack limit"]
owner: devops-agent
objective: "Resolve stack-based coverage issues"
context: "CloudFormation stacks can be used to define monitoring scope."
---

## Phase 1 — Triage

MUST:
- List monitored stacks
- Verify stack exists and is in valid state
- Check stack resource count
- Verify stack limit not exceeded

SHOULD:
- Check for nested stacks
- Verify stack resources are supported services

## Phase 2 — Remediate

MUST:
- Add stacks to coverage
- Fix stack state issues
- Use tag-based coverage if stack limit reached

## Common Issues

- symptoms: "Stack added but resources not monitored"
  diagnosis: "Stack resources are unsupported services."
  resolution: "Verify stack contains supported service resources."

## Output Format

```yaml
root_cause: "stack_config — <specific_cause>"
evidence:
  - type: stack_coverage
    content: "<stack configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix stack coverage"
  long_term: "Use tag-based coverage for flexibility"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Stack coverage is operational configuration |
| Blast Radius | MEDIUM — Stack coverage changes affect monitoring for all stack resources |
| Rollback Complexity | LOW — Stack coverage can be updated |
| Compliance Scope | LOW — Stack configuration is operational |

## Escalation Conditions

- Escalate to operations if stack coverage limits are reached
- Escalate to CloudFormation owners if stack state issues affect coverage

## Data Sensitivity

- **Stack identifiers** are standard infrastructure metadata
- **Stack resource lists** reveal infrastructure composition

## PROHIBITED Actions

- Do NOT remove stacks from coverage without understanding the monitoring impact
- Do NOT exceed stack limits without switching to tag-based coverage

## Phase 3 — Rollback

- Re-add stacks that were removed from coverage
- Switch to tag-based coverage if stack limits are reached

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
