---
title: "C2 — Security Finding Management"
description: "Diagnose security finding triage and remediation"
status: active
severity: LOW
triggers:
  - "false positive security finding"
  - "how to fix finding"
  - "finding severity unclear"
owner: devops-agent
objective: "Resolve security finding management issues"
context: "Security findings include severity, description, and remediation guidance."
---

## Phase 1 — Triage

MUST:
- Review finding details and severity
- Determine if true or false positive
- Check remediation guidance
- Verify affected code location

SHOULD:
- Cross-reference with other security tools
- Check finding confidence level

## Phase 2 — Remediate

MUST:
- Fix true positive findings
- Suppress confirmed false positives with rationale

SHOULD:
- Prioritize by severity
- Track remediation progress

## Common Issues

- symptoms: "Many low-confidence security findings"
  diagnosis: "Detector flagging patterns that may not be vulnerabilities."
  resolution: "Review in context. Suppress false positives. Focus on high-confidence findings."

## Output Format

```yaml
root_cause: "security_findings — <specific_cause>"
evidence:
  - type: findings
    content: "<finding details>"
severity: LOW
mitigation:
  immediate: "Triage findings"
  long_term: "Establish finding review process"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix true positive findings | YELLOW |
| Suppress confirmed false positives with rationale | GREEN |
| Prioritize by severity | GREEN |
| Track remediation progress | GREEN |

## Escalation Conditions

- Critical or high-severity true positive findings require immediate attention
- Large number of unaddressed security findings accumulating

## Data Sensitivity

- **HIGH**: security findings expose specific vulnerabilities, secrets, and code weaknesses
- Finding details may contain sensitive code patterns and remediation guidance

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If findings were suppressed incorrectly, unsuppress them
2. If code fixes introduced regressions, revert code changes and re-address findings
3. Suppression rationale should be documented for audit purposes
4. Verify rollback by confirming finding status matches expected triage state

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
