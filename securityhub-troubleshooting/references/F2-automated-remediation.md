---
title: "F2 — Automated Remediation"
description: "Diagnose Security Hub automated remediation and automation rule issues"
status: active
severity: MEDIUM
triggers:
  - "automated remediation"
  - "automation rule"
  - "auto-remediate"
  - "finding automation"
  - "automatic fix"
owner: devops-agent
objective: "Identify and resolve automated remediation and automation rule issues"
context: "Security Hub supports automated remediation through automation rules (native) and custom actions with EventBridge (custom). Automation rules can update finding fields, suppress findings, or trigger workflows. Custom remediation uses EventBridge + Lambda/SSM for actual resource changes."
---

## Phase 1 — Triage

MUST:
- List automation rules: `aws securityhub list-automation-rules`
- Get automation rule details: `aws securityhub batch-get-automation-rules --automation-rules-arns <rule-arns>`
- Check if the rule criteria match the intended findings
- Verify the rule actions are correct (update fields, suppress, etc.)
- Check the rule status (ENABLED/DISABLED)

SHOULD:
- Check EventBridge rules for custom remediation: `aws events list-rules`
- Verify Lambda functions used for remediation: `aws lambda list-functions`
- Check SSM Automation documents for remediation: `aws ssm list-documents --filters Key=DocumentType,Values=Automation`

MAY:
- Check CloudTrail for automation rule execution events
- Review Lambda CloudWatch logs for remediation errors
- Check SSM Automation execution history

## Phase 2 — Remediate

MUST:
- Fix automation rule criteria to match intended findings
- Verify rule actions produce the desired result
- Enable the rule if disabled

SHOULD:
- Test automation rules with sample findings before enabling broadly
- Set up monitoring for automation rule execution
- Implement error handling in Lambda remediation functions

MAY:
- Create automation rules for common finding patterns
- Document remediation workflows

## Common Issues

- symptoms: "Automation rule created but findings not being updated"
  diagnosis: "The rule criteria do not match the findings, or the rule is disabled."
  resolution: "Verify the rule criteria match the finding attributes. Enable the rule."

- symptoms: "Automation rule matches too many findings"
  diagnosis: "The rule criteria are too broad, affecting findings that should not be automated."
  resolution: "Narrow the rule criteria with additional conditions (severity, finding type, resource type)."

- symptoms: "Custom remediation Lambda function fails"
  diagnosis: "The Lambda function has errors, lacks permissions, or cannot handle the event format."
  resolution: "Check Lambda logs, fix errors, and ensure the function has permissions for the remediation actions."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix automation rule criteria | YELLOW | Rule modification; reversible |
| Enable automation rule | YELLOW | State change; reversible |
| Fix Lambda remediation | YELLOW | Code change; previous version restorable |

## Escalation Conditions

- Cross-account configuration changes
- Automation rule changes affecting finding management
- Remediation action deployment

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-automation-rules` | LOW | Rule names and ARNs |
| `batch-get-automation-rules` | MEDIUM | Rule criteria and actions |
| Lambda logs | HIGH | Remediation execution details |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Automation rule fix | Revert via `batch-update-automation-rules` |
| Rule enablement | Disable via `batch-update-automation-rules` |
| Lambda code fix | Redeploy previous Lambda version |

## Output Format

```yaml
root_cause: "automated_remediation — <specific_cause>"
evidence:
  - type: automation_rules
    content: "<automation rule configuration>"
  - type: eventbridge
    content: "<EventBridge rules for remediation>"
  - type: execution
    content: "<remediation execution results>"
severity: MEDIUM
mitigation:
  immediate: "Fix the automation rule or remediation configuration"
  long_term: "Test automation thoroughly and monitor execution"
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
