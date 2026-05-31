---
title: "A2 — Assessment Configuration"
description: "Diagnose assessment configuration issues"
status: active
severity: MEDIUM
triggers: ["assessment config wrong", "scope incorrect", "assessment settings"]
owner: devops-agent
objective: "Resolve assessment configuration issues"
context: "Assessment configuration includes scope, roles, and report destination. Misconfiguration affects evidence collection and reporting."
---

## Phase 1 — Triage

MUST:
- Get assessment details: `aws auditmanager get-assessment --assessment-id <id>`
- Verify scope includes correct accounts and services
- Check assessment roles and S3 destination

SHOULD:
- Verify assessment status is ACTIVE
- Check if assessment was recently modified

MAY:
- Compare with other assessments for consistency

## Phase 2 — Remediate

MUST:
- Update scope if incorrect: `aws auditmanager update-assessment --assessment-id <id> --scope <scope>`
- Fix S3 destination if reports failing

SHOULD:
- Review scope after organizational changes

MAY:
- Automate scope updates with Lambda

## Common Issues

- symptoms: "Assessment missing resources from new account"
  diagnosis: "New account not in assessment scope."
  resolution: "Update assessment scope to include the new account."

## Output Format

```yaml
root_cause: "assessment_configuration — <specific_cause>"
evidence:
  - type: assessment_config
    content: "<scope and configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration"
  long_term: "Automate scope management"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update scope if incorrect | YELLOW | Changes what resources are assessed — affects compliance coverage |
| Fix S3 destination if reports failing | GREEN | Configuration fix — non-destructive |
| Review scope after organizational changes | GREEN | Read-only review |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Scope changes may create gaps in compliance coverage
- Configuration changes affect active audit periods

## Data Sensitivity

- Assessment scope reveals which accounts and services are under compliance monitoring
- Configuration details reveal audit methodology and coverage
- Scope changes may expose or hide resources from compliance evaluation

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER reduce assessment scope without auditor approval
- NEVER modify assessment configuration during active audit periods without documentation

## Phase 3 — Rollback

- If scope was changed incorrectly: restore previous scope using `aws auditmanager update-assessment`
- If S3 destination was changed: revert to original destination
- Document all scope and configuration changes for compliance audit trail

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
