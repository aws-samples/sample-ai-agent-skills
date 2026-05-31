---
title: "C1 — Control Evaluation"
description: "Diagnose control evaluation issues"
status: active
severity: HIGH
triggers: ["control not evaluating", "no evidence for control", "control inactive"]
owner: devops-agent
objective: "Resolve control evaluation issues"
context: "Controls evaluate compliance by collecting evidence from various sources (Config, CloudTrail, Security Hub, manual). Evaluation depends on the data source being available."
---

## Phase 1 — Triage

MUST:
- Check control status in assessment: `aws auditmanager get-assessment --assessment-id <id>`
- Verify evidence source is enabled (Config, Security Hub, CloudTrail)
- Check evidence folders for the control

SHOULD:
- Verify Config rules referenced by the control are active
- Check if evidence collection schedule has run

MAY:
- Compare with controls that are working correctly

## Phase 2 — Remediate

MUST:
- Enable required evidence source (Config, Security Hub)
- Ensure Config rules are active and evaluating

SHOULD:
- Wait for next evidence collection cycle (up to 24 hours)

MAY:
- Manually trigger Config rule evaluation: `aws config start-config-rules-evaluation --config-rule-names <rule>`

## Common Issues

- symptoms: "Control shows 0 evidence items"
  diagnosis: "AWS Config not enabled or Config rule not active."
  resolution: "Enable Config and activate the required Config rule."

## Output Format

```yaml
root_cause: "control_evaluation — <specific_cause>"
evidence:
  - type: control_status
    content: "<control evaluation status and evidence source>"
severity: HIGH
mitigation:
  immediate: "Enable required evidence source"
  long_term: "Verify all evidence sources before creating assessments"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable required evidence source (Config, Security Hub) | YELLOW | Enables AWS services — may incur costs and affect account |
| Ensure Config rules are active and evaluating | YELLOW | Activates Config rules — may trigger evaluations |
| Wait for next evidence collection cycle | GREEN | No action — passive waiting |
| Manually trigger Config rule evaluation | GREEN | Triggers existing rule — non-destructive |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Enabling AWS Config or Security Hub affects account-wide monitoring
- Config rule activation may flag existing non-compliant resources

## Data Sensitivity

- Control evaluation results reveal compliance posture of assessed resources
- Evidence collection may capture sensitive configuration data
- Config rule evaluations may reveal security vulnerabilities

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER disable evidence sources for active assessments
- NEVER modify Config rules that are shared across compliance frameworks

## Phase 3 — Rollback

- If AWS Config was enabled incorrectly: disable the Config recorder (note: stops all Config-based evidence)
- If Config rules were activated incorrectly: deactivate the specific rules
- Document all evidence source changes for compliance audit trail

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
