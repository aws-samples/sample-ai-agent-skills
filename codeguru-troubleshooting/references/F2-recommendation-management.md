---
title: "F2 — Recommendation Management"
description: "Diagnose recommendation tracking and workflow"
status: active
severity: LOW
triggers:
  - "too many recommendations"
  - "recommendation priority"
  - "recommendation tracking"
owner: devops-agent
objective: "Resolve recommendation management issues"
context: "Recommendations have severity and confidence levels for prioritization."
---

## Phase 1 — Triage

MUST:
- List recommendations: `aws codeguru-reviewer list-recommendations --code-review-arn <arn>`
- Check recommendation severity distribution
- Review confidence levels
- Check for actionable vs informational

SHOULD:
- Prioritize by severity and confidence
- Group by recommendation type

## Phase 2 — Remediate

MUST:
- Address high-severity, high-confidence findings first
- Suppress confirmed false positives

SHOULD:
- Track recommendation trends
- Measure fix rate

## Common Issues

- symptoms: "Overwhelmed by number of recommendations"
  diagnosis: "No prioritization strategy."
  resolution: "Filter by severity=Critical/High. Focus on high-confidence. Suppress false positives."

## Output Format

```yaml
root_cause: "recommendation_mgmt — <specific_cause>"
evidence:
  - type: recommendations
    content: "<recommendation summary>"
severity: LOW
mitigation:
  immediate: "Prioritize and triage"
  long_term: "Establish review workflow"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Address high-severity, high-confidence findings first | GREEN |
| Suppress confirmed false positives | GREEN |
| Track recommendation trends | GREEN |
| Measure fix rate | GREEN |

## Escalation Conditions

- Large backlog of unaddressed high-severity recommendations
- Fix rate declining over time indicating process issues

## Data Sensitivity

- **MEDIUM**: recommendation data reveals code quality trends and security posture over time
- Recommendation management patterns expose development team practices

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If recommendations were suppressed incorrectly, unsuppress them
2. If prioritization criteria were changed, restore previous triage process
3. Recommendation data is read-only — no destructive rollback needed
4. Verify rollback by confirming recommendation triage state matches expected configuration

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
