---
title: "D1 — Repository Association Failures"
description: "Diagnose repository association creation and state issues"
status: active
severity: HIGH
triggers:
  - "association failed"
  - "cannot associate repository"
  - "association stuck"
owner: devops-agent
objective: "Resolve repository association failures"
context: "Repository association connects CodeGuru to source repositories."
---

## Phase 1 — Triage

MUST:
- Check association state: `aws codeguru-reviewer list-repository-associations`
- Verify provider connection (CodeStar Connection for GitHub)
- Check IAM permissions
- Review association error

SHOULD:
- Verify repository exists and is accessible
- Check CodeStar Connection status
- Verify provider-specific requirements

## Phase 2 — Remediate

MUST:
- Fix provider connection
- Update IAM permissions
- Recreate association if FAILED

SHOULD:
- Test connection before associating
- Document association requirements

## Common Issues

- symptoms: "Association stuck in ASSOCIATING"
  diagnosis: "CodeStar Connection not authorized."
  resolution: "Complete CodeStar Connection authorization in console."

## Output Format

```yaml
root_cause: "repository_association — <specific_cause>"
evidence:
  - type: association_status
    content: "<association state>"
severity: HIGH
mitigation:
  immediate: "Fix association"
  long_term: "Automate association management"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix provider connection | YELLOW |
| Update IAM permissions | YELLOW |
| Recreate association if FAILED | GREEN |
| Test connection before associating | GREEN |
| Document association requirements | GREEN |

## Escalation Conditions

- Fix requires modifying production CodeStar Connection configurations
- Fix involves persistent association failures blocking code reviews

## Data Sensitivity

- **MEDIUM**: repository associations expose source code repository locations and provider configurations
- CodeStar Connection details reveal integration architecture

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If association was recreated, delete the incorrect one: `aws codeguru-reviewer disassociate-repository --association-arn <arn>`
2. If IAM permissions were modified, revert to previous policy
3. If CodeStar Connection was changed, restore previous connection configuration
4. Verify rollback by confirming `aws codeguru-reviewer list-repository-associations` shows expected state

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
