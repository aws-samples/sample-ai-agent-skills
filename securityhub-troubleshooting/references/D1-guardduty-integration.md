---
title: "D1 — GuardDuty Integration"
description: "Diagnose Security Hub integration with Amazon GuardDuty"
status: active
severity: MEDIUM
triggers:
  - "guardduty integration"
  - "guardduty findings security hub"
  - "guardduty not sending"
  - "guardduty product"
  - "threat findings missing"
owner: devops-agent
objective: "Identify and resolve GuardDuty finding integration issues with Security Hub"
context: "GuardDuty automatically sends findings to Security Hub when both services are enabled in the same region. The integration is automatic but can be disabled. Suppressed GuardDuty findings are not sent to Security Hub."
---

## Phase 1 — Triage

MUST:
- Verify GuardDuty integration is enabled: `aws securityhub list-enabled-products-for-import`
- Check GuardDuty is enabled: `aws guardduty list-detectors`
- Get GuardDuty findings in Security Hub: `aws securityhub get-findings --filters '{"ProductName":[{"Value":"GuardDuty","Comparison":"EQUALS"}]}' --max-results 10`
- Verify both services are in the same region
- Check GuardDuty detector status: `aws guardduty get-detector --detector-id <detector-id>`

SHOULD:
- Check for recent GuardDuty findings: `aws guardduty list-findings --detector-id <detector-id> --max-results 5`
- Verify GuardDuty suppression rules are not hiding all findings
- Check Security Hub finding count from GuardDuty

MAY:
- Check CloudTrail for integration events
- Verify cross-region aggregation if expecting findings from other regions

## Phase 2 — Remediate

MUST:
- Enable GuardDuty if not enabled
- Enable the GuardDuty product integration in Security Hub
- Verify findings start flowing after enabling

SHOULD:
- Enable both services in all required regions
- Set up cross-region aggregation for centralized view

MAY:
- Create automation rules for GuardDuty finding response
- Set up custom actions for GuardDuty findings

## Common Issues

- symptoms: "GuardDuty findings not appearing in Security Hub"
  diagnosis: "GuardDuty is not enabled, the integration is disabled, or services are in different regions."
  resolution: "Enable both services in the same region. Verify GuardDuty appears in list-enabled-products-for-import."

- symptoms: "Only some GuardDuty findings appear in Security Hub"
  diagnosis: "Suppressed (auto-archived) GuardDuty findings are not sent to Security Hub."
  resolution: "Review GuardDuty suppression rules. Suppressed findings are excluded from Security Hub export."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable GuardDuty | YELLOW | Service enablement; reversible |
| Enable product integration | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Service enablement decisions

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-enabled-products-for-import` | LOW | Integration list |
| `get-detector` | MEDIUM | Detector configuration |
| `get-findings` (GuardDuty) | HIGH | Threat findings |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| GuardDuty enablement | Disable detector (not recommended) |
| Integration enablement | Disable via `disable-import-findings-for-product` |

## Output Format

```yaml
root_cause: "guardduty_integration — <specific_cause>"
evidence:
  - type: integration_status
    content: "<product integration status>"
  - type: guardduty
    content: "<GuardDuty detector and finding status>"
severity: MEDIUM
mitigation:
  immediate: "Enable the integration and verify finding flow"
  long_term: "Monitor integration health and finding counts"
```

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
