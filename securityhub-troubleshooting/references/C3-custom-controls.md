---
title: "C3 — Custom Control Issues"
description: "Diagnose custom security control configuration and evaluation issues"
status: active
severity: MEDIUM
triggers:
  - "custom control"
  - "custom standard"
  - "custom security check"
  - "custom policy"
  - "custom finding"
owner: devops-agent
objective: "Identify and resolve custom control and custom standard configuration issues"
context: "Security Hub supports custom security controls through custom Config rules that generate findings in ASFF format. Custom controls can be associated with custom standards or used independently. Issues arise from Config rule configuration, ASFF format, or control association."
---

## Phase 1 — Triage

MUST:
- Check custom Config rules generating Security Hub findings: `aws configservice describe-config-rules`
- Verify the custom rule is evaluating: `aws configservice describe-config-rule-evaluation-status`
- Check if findings are being generated in ASFF format
- Verify the findings appear in Security Hub: `aws securityhub get-findings --filters '{"ProductName":[{"Value":"Security Hub","Comparison":"EQUALS"}]}' --max-results 10`

SHOULD:
- Check automation rules that may affect custom findings: `aws securityhub list-automation-rules`
- Verify the custom rule's Lambda function (if custom rule): `aws lambda get-function --function-name <function-name>`
- Check for ASFF format errors in finding generation

MAY:
- Review the custom control's evaluation logic
- Check for conflicts with built-in controls

## Phase 2 — Remediate

MUST:
- Fix Config rule evaluation issues
- Ensure findings conform to ASFF format
- Verify custom controls produce expected compliance results

SHOULD:
- Test custom rules with known compliant and non-compliant resources
- Document custom control requirements and evaluation logic

MAY:
- Create a custom standard to group related custom controls
- Set up monitoring for custom control evaluation

## Common Issues

- symptoms: "Custom Config rule evaluates but findings not in Security Hub"
  diagnosis: "The Config rule findings are not being sent to Security Hub, or the integration is not configured."
  resolution: "Ensure the Config integration is enabled in Security Hub. Config rule compliance results automatically flow to Security Hub."

- symptoms: "Custom control shows incorrect compliance status"
  diagnosis: "The custom Config rule logic does not correctly evaluate the compliance condition."
  resolution: "Review and fix the custom rule's evaluation logic. Test with known resources."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix Config rule evaluation | YELLOW | Rule fix; reversible |
| Fix custom rule Lambda | YELLOW | Code change; previous version restorable |

## Escalation Conditions

- Cross-account configuration changes
- Custom control changes affecting compliance reporting

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-config-rules` | LOW | Rule configuration |
| `get-findings` (custom) | MEDIUM | Custom control findings |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Config rule fix | Revert rule configuration |
| Lambda code fix | Redeploy previous Lambda version |

## Output Format

```yaml
root_cause: "custom_control — <specific_cause>"
evidence:
  - type: config_rule
    content: "<custom Config rule status>"
  - type: findings
    content: "<custom control findings>"
severity: MEDIUM
mitigation:
  immediate: "Fix the custom control configuration"
  long_term: "Test and document custom controls thoroughly"
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
