---
title: "C2 — Disabled Controls"
description: "Diagnose issues with disabled Security Hub controls and re-enablement"
status: active
severity: MEDIUM
triggers:
  - "disabled control"
  - "control disabled"
  - "re-enable control"
  - "control not running"
  - "control turned off"
owner: devops-agent
objective: "Identify and resolve issues with disabled controls and manage control enablement"
context: "Controls can be disabled to exclude them from compliance scoring and stop new finding generation. Existing findings from disabled controls remain. Re-enabling triggers fresh evaluation. In multi-account setups, the admin can control which controls are enabled for members."
---

## Phase 1 — Triage

MUST:
- List controls and their enabled/disabled status: `aws securityhub describe-standards-controls --standards-subscription-arn <standard-arn>`
- Identify which controls are disabled and the disable reason
- Check if the control was disabled by the admin or locally
- Verify existing findings from disabled controls: `aws securityhub get-findings --filters '{"ComplianceSecurityControlId":[{"Value":"<control-id>","Comparison":"EQUALS"}]}' --max-results 10`

SHOULD:
- Check CloudTrail for control disable events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateStandardsControl --max-results 10`
- Verify if the account is managed by an admin: `aws securityhub get-administrator-account`
- Check organization-level control settings

MAY:
- Review the justification for disabling each control
- Check if disabled controls affect compliance score requirements

## Phase 2 — Remediate

MUST:
- Re-enable controls that should be active: `aws securityhub update-standards-control --standards-control-arn <control-arn> --control-status ENABLED`
- Verify the control starts evaluating after re-enablement
- Archive existing findings from controls that should remain disabled

SHOULD:
- Document the justification for any controls that remain disabled
- Verify Config is enabled for re-enabled controls to evaluate

MAY:
- Create a control enablement policy for the organization
- Set up monitoring for control status changes

## Common Issues

- symptoms: "Cannot re-enable a control — AccessDeniedException"
  diagnosis: "The account is managed by an admin who controls the control settings, or IAM permissions are insufficient."
  resolution: "Contact the admin to re-enable the control, or add securityhub:UpdateStandardsControl permission."

- symptoms: "Control re-enabled but shows NO_DATA"
  diagnosis: "Config is not recording the required resource type, or the control needs time for initial evaluation."
  resolution: "Verify Config coverage and wait for evaluation (up to 24 hours for initial results)."

- symptoms: "Findings from disabled control still appearing in dashboard"
  diagnosis: "Disabling a control does not delete existing findings. They remain until archived or aged out."
  resolution: "Manually archive findings from disabled controls using batch-update-findings."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Re-enable controls | YELLOW | Enables evaluation; reversible |
| Archive findings from disabled controls | YELLOW | State change; findings can be unarchived |

## Escalation Conditions

- Standard enablement/disablement
- Control enablement/disablement decisions
- Cross-account configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-standards-controls` | LOW | Control status |
| `get-findings` (control) | MEDIUM | Compliance findings |
| CloudTrail events | MEDIUM | Control change history |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings
- NEVER disable controls to improve compliance score

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Control re-enablement | Disable via `update-standards-control` |
| Finding archival | Unarchive findings |

## Output Format

```yaml
root_cause: "disabled_control — <specific_cause>"
evidence:
  - type: control_status
    content: "<control enabled/disabled status>"
  - type: findings
    content: "<existing findings from disabled controls>"
severity: MEDIUM
mitigation:
  immediate: "Re-enable the control or archive existing findings"
  long_term: "Document control enablement policy and monitor changes"
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
