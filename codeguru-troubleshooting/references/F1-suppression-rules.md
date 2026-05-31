---
title: "F1 — Suppression Rules Issues"
description: "Diagnose recommendation suppression configuration"
status: active
severity: LOW
triggers:
  - "suppression not working"
  - "recommendation keeps appearing"
  - "cannot suppress finding"
owner: devops-agent
objective: "Resolve suppression rule issues"
context: "Suppressions hide recommendations but don't delete them."
---

## Phase 1 — Triage

MUST:
- Check suppression rule configuration
- Verify rule matches the recommendation
- Check rule scope (file, pattern, recommendation type)

SHOULD:
- Review suppression history
- Check for conflicting rules

## Phase 2 — Remediate

MUST:
- Fix suppression rule matching
- Correct rule scope
- Verify suppression takes effect

SHOULD:
- Document suppression rationale
- Review suppressions periodically

## Common Issues

- symptoms: "Suppressed recommendation reappears on new PR"
  diagnosis: "Suppression scope too narrow."
  resolution: "Broaden suppression rule to cover pattern, not just specific instance."

## Output Format

```yaml
root_cause: "suppression — <specific_cause>"
evidence:
  - type: suppression_config
    content: "<suppression rules>"
severity: LOW
mitigation:
  immediate: "Fix suppression rules"
  long_term: "Review suppressions quarterly"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix suppression rule matching | GREEN |
| Correct rule scope | GREEN |
| Verify suppression takes effect | GREEN |
| Document suppression rationale | GREEN |
| Review suppressions periodically | GREEN |

## Escalation Conditions

- Fix involves broad suppression rules that may hide real security issues
- Suppression rules affecting critical or high-severity findings

## Data Sensitivity

- **MEDIUM**: suppression rules reveal which findings are being intentionally ignored
- Suppression rationale documents risk acceptance decisions

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If suppression rules were modified incorrectly, restore previous rule configuration
2. If scope was broadened too much, narrow it back to the intended scope
3. Unsuppress any findings that were incorrectly suppressed
4. Verify rollback by confirming suppression rules match expected configuration

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
