---
title: "D2 — Repository Provider Configuration"
description: "Diagnose provider-specific configuration issues"
status: active
severity: MEDIUM
triggers:
  - "GitHub connection error"
  - "CodeCommit access denied"
  - "Bitbucket integration"
owner: devops-agent
objective: "Resolve provider configuration issues"
context: "Each provider (GitHub, CodeCommit, Bitbucket) has different requirements."
---

## Phase 1 — Triage

MUST:
- Identify repository provider
- Check provider-specific connection
- Verify credentials and permissions
- Check for provider API rate limits

SHOULD:
- Test provider connectivity independently
- Verify webhook configuration

## Phase 2 — Remediate

MUST:
- Fix provider credentials
- Complete connection authorization
- Update permissions

SHOULD:
- Document provider setup steps
- Monitor connection health

## Common Issues

- symptoms: "GitHub PR reviews not triggering"
  diagnosis: "CodeStar Connection webhook not configured."
  resolution: "Verify CodeStar Connection. Check GitHub app installation."

## Output Format

```yaml
root_cause: "provider_config — <specific_cause>"
evidence:
  - type: provider_status
    content: "<provider configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix provider configuration"
  long_term: "Document provider setup"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix provider credentials | YELLOW |
| Complete connection authorization | GREEN |
| Update permissions | YELLOW |
| Document provider setup steps | GREEN |
| Monitor connection health | GREEN |

## Escalation Conditions

- Fix requires modifying production provider credentials or connections
- Fix involves persistent provider connectivity issues

## Data Sensitivity

- **MEDIUM**: provider configurations contain connection credentials and repository access details
- Webhook configurations reveal CI/CD integration architecture

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If provider credentials were changed, restore previous credentials
2. If connection authorization was modified, verify it still has correct repository access
3. If webhook configuration was changed, restore previous settings
4. Verify rollback by confirming provider connectivity and review triggers work correctly

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
