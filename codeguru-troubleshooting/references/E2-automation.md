---
title: "E2 — Review Automation Issues"
description: "Diagnose automated review scheduling and triggers"
status: active
severity: LOW
triggers:
  - "scheduled scan not running"
  - "automation error"
  - "trigger not firing"
owner: devops-agent
objective: "Resolve review automation issues"
context: "Automated reviews can be scheduled or triggered by events."
---

## Phase 1 — Triage

MUST:
- Check automation configuration
- Verify EventBridge rules if using event triggers
- Check IAM permissions for automation
- Review execution history

SHOULD:
- Check schedule expression syntax
- Verify target configuration

## Phase 2 — Remediate

MUST:
- Fix automation configuration
- Correct IAM permissions
- Fix schedule expressions

SHOULD:
- Test automation manually first
- Monitor automation execution

## Common Issues

- symptoms: "Scheduled full scan not executing"
  diagnosis: "EventBridge rule disabled or IAM permission missing."
  resolution: "Enable EventBridge rule. Fix IAM permissions."

## Output Format

```yaml
root_cause: "automation — <specific_cause>"
evidence:
  - type: automation_config
    content: "<automation configuration>"
severity: LOW
mitigation:
  immediate: "Fix automation"
  long_term: "Monitor automation health"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix automation configuration | GREEN |
| Correct IAM permissions | YELLOW |
| Fix schedule expressions | GREEN |
| Test automation manually first | GREEN |
| Monitor automation execution | GREEN |

## Escalation Conditions

- Fix requires modifying production EventBridge rules or automation configurations
- Fix involves persistent automation failures affecting review coverage

## Data Sensitivity

- **LOW**: automation configurations expose scheduling patterns and trigger mechanisms
- Execution history reveals review frequency and coverage patterns

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If EventBridge rule was modified, restore previous rule configuration
2. If IAM permissions were changed, revert to previous policy
3. If schedule expression was changed, restore previous schedule
4. Verify rollback by confirming automation executes on expected schedule

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
