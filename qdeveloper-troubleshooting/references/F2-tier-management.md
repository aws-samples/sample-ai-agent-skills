---
title: "F2 — Tier Management Issues"
description: "Diagnose subscription tier and billing problems"
status: active
severity: LOW
triggers:
  - "upgrade to Pro"
  - "billing error"
  - "tier not changing"
owner: devops-agent
objective: "Resolve tier management issues"
context: "Pro tier requires IAM Identity Center and organization setup."
---

## Phase 1 — Triage

MUST:
- Check current subscription tier
- Verify IAM Identity Center for Pro tier
- Check billing configuration
- Verify user assignment

SHOULD:
- Review Pro tier requirements
- Check organization admin settings

## Phase 2 — Remediate

MUST:
- Configure IAM Identity Center for Pro
- Complete subscription setup
- Assign users to Pro tier

SHOULD:
- Document tier management process
- Monitor per-user costs

## Common Issues

- symptoms: "Cannot upgrade to Pro tier"
  diagnosis: "IAM Identity Center not configured."
  resolution: "Set up IAM Identity Center. Configure Q Developer Pro subscription."

## Output Format

```yaml
root_cause: "tier_management — <specific_cause>"
evidence:
  - type: subscription
    content: "<tier and billing details>"
severity: LOW
mitigation:
  immediate: "Fix subscription setup"
  long_term: "Document tier management"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — tier management is an administrative concern
- Service Disruption Risk: LOW — tier issues affect feature availability
- Compliance Impact: LOW — tier management is an operational concern

## Escalation Conditions
- Tier upgrade/downgrade fails
- Feature access does not match tier level
- Billing issues related to tier changes
- Multiple users affected by tier issues
- Tier changes not propagating

## Data Sensitivity
- Tier configurations — INTERNAL
- Billing information — CONFIDENTIAL
- User assignments — INTERNAL
- Feature access details — INTERNAL
- Never expose billing details

## Prohibited Actions
- Do NOT downgrade tiers without notifying affected users
- Do NOT bypass tier change approval processes
- Do NOT ignore tier-related billing discrepancies
- Do NOT share tier configurations externally
- Do NOT modify tier assignments without authorization
- Do NOT ignore feature access issues

## Phase 3 — Rollback
- Restore previous tier assignments
- Revert tier configuration changes
- Verify feature access after changes
- Restore billing configurations
- Document tier issues
- Update tier management procedures

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
