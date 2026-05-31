---
title: "C1 — Control Evaluation Failures"
description: "Diagnose Security Hub control evaluation failures and NO_DATA issues"
status: active
severity: HIGH
triggers:
  - "control failed"
  - "control NO_DATA"
  - "control not evaluating"
  - "security check failed"
  - "control UNKNOWN"
owner: devops-agent
objective: "Identify and resolve Security Hub control evaluation failures"
context: "Security Hub controls evaluate underlying Config rules. Failures can be caused by Config not being enabled, Config rules not evaluating, resource type not recorded, or the control being in an error state. NO_DATA typically indicates a Config dependency issue."
---

## Phase 1 — Triage

MUST:
- Check the control status: `aws securityhub describe-standards-controls --standards-subscription-arn <standard-arn>`
- Verify Config is enabled: `aws configservice describe-configuration-recorder-status`
- Check Config rule evaluation for the backing rule: `aws configservice describe-config-rule-evaluation-status`
- Get findings for the specific control: `aws securityhub get-findings --filters '{"ComplianceSecurityControlId":[{"Value":"<control-id>","Comparison":"EQUALS"}]}' --max-results 10`
- Verify Config records the resource type the control evaluates

SHOULD:
- Check the Config rule compliance: `aws configservice describe-compliance-by-config-rule`
- Verify the Config delivery channel is working: `aws configservice describe-delivery-channel-status`
- Check for Config rule errors in CloudWatch

MAY:
- Force Config rule re-evaluation: `aws configservice start-config-rules-evaluation --config-rule-names <rule-name>`
- Check if the control was recently enabled (may need time for initial evaluation)

## Phase 2 — Remediate

MUST:
- Enable Config recorder if not running
- Ensure Config records the required resource types
- Fix any Config rule evaluation errors

SHOULD:
- Wait for initial evaluation after enabling Config (up to 24 hours)
- Verify the control produces findings after fixing Config

MAY:
- Set up monitoring for control evaluation status
- Create alerts for controls that transition to NO_DATA

## Common Issues

- symptoms: "Control shows NO_DATA for all resources"
  diagnosis: "Config is not enabled, not recording the required resource type, or the Config rule is not evaluating."
  resolution: "Enable Config with the required resource types. Verify the backing Config rule is evaluating."

- symptoms: "Control shows FAILED but the resource appears compliant"
  diagnosis: "The control may evaluate different attributes than expected, or Config has stale data."
  resolution: "Review the control's evaluation criteria. Force Config rule re-evaluation."

- symptoms: "Control was passing but suddenly shows NO_DATA"
  diagnosis: "Config recorder was stopped, the Config rule was deleted, or the delivery channel failed."
  resolution: "Check Config recorder status and delivery channel. Restart if needed."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Config recorder | YELLOW | Service enablement; reversible |
| Fix Config rule evaluation | YELLOW | Rule fix; reversible |
| Force re-evaluation | GREEN | Triggers evaluation; no destructive change |

## Escalation Conditions

- Standard enablement/disablement
- Config recorder changes
- Cross-account configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-standards-controls` | LOW | Control status |
| `describe-configuration-recorder-status` | LOW | Recorder status |
| `get-findings` (control) | MEDIUM | Compliance findings |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Config recorder enablement | Stop recorder (not recommended) |
| Config rule fix | Revert rule configuration |

## Output Format

```yaml
root_cause: "control_failure — <specific_cause>"
evidence:
  - type: control_status
    content: "<control evaluation status>"
  - type: config_status
    content: "<Config recorder and rule status>"
  - type: findings
    content: "<control findings>"
severity: HIGH
mitigation:
  immediate: "Fix Config dependency and verify control evaluation"
  long_term: "Monitor Config and control evaluation status continuously"
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
