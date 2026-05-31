---
title: "D1 — Auto-Remediation Failures"
description: "Diagnose AWS Config automatic remediation action failures"
status: active
severity: HIGH
triggers:
  - "remediation failed"
  - "auto-remediation not working"
  - "remediation action error"
  - "automatic fix failed"
  - "remediation not executing"
owner: devops-agent
objective: "Identify and resolve Config auto-remediation failures to restore automated compliance enforcement"
context: "Config auto-remediation uses SSM Automation documents to fix non-compliant resources. Failures can occur due to IAM permission issues, SSM document errors, resource state conflicts, concurrency limits, or incorrect remediation parameters."
---

## Phase 1 — Triage

MUST:
- Check remediation configuration: `aws configservice describe-remediation-configurations --config-rule-names <rule-name>`
- Check remediation execution status: `aws configservice describe-remediation-execution-status --config-rule-name <rule-name>`
- Verify the SSM Automation document exists: `aws ssm describe-document --name <document-name>`
- Check SSM Automation execution history: `aws ssm describe-automation-executions --filters Key=DocumentNamePrefix,Values=<document-name>`
- Verify the remediation execution role: `aws iam get-role --role-name <remediation-role>`

SHOULD:
- Get detailed SSM execution steps: `aws ssm get-automation-execution --automation-execution-id <execution-id>`
- Check CloudTrail for remediation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20`
- Verify the remediation role has permissions for the target service

MAY:
- Check remediation retry settings and concurrency limits
- Test the SSM Automation document manually: `aws ssm start-automation-execution --document-name <document-name> --parameters '<params>'`

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for the remediation execution role
- Correct SSM Automation document parameters
- Verify the target resource is in a state that allows remediation

SHOULD:
- Test the remediation manually before re-enabling auto-remediation
- Update concurrency and error threshold settings if needed
- Ensure the SSM document version matches the remediation configuration

MAY:
- Create custom SSM Automation documents for complex remediation scenarios
- Set up CloudWatch alarms for remediation failures

## Common Issues

- symptoms: "Remediation execution shows FAILED with AccessDenied"
  diagnosis: "The remediation execution role lacks permissions for the target service actions."
  resolution: "Add the required service permissions to the remediation execution role."

- symptoms: "Remediation configured but never executes"
  diagnosis: "Auto-remediation is not enabled, or the rule has no NON_COMPLIANT resources."
  resolution: "Verify automatic remediation is enabled in the remediation configuration and the rule has non-compliant resources."

- symptoms: "Remediation executes but resource remains non-compliant"
  diagnosis: "The SSM Automation document does not correctly fix the compliance issue, or the rule re-evaluates before the fix takes effect."
  resolution: "Review the SSM document logic, verify it addresses the specific compliance check, and allow time for re-evaluation."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix IAM role permissions | YELLOW | Permission change; reversible |
| Fix SSM document parameters | YELLOW | Parameter change; reversible |
| Re-run remediation | YELLOW | Executes SSM automation; may change resources |

## Escalation Conditions

- Remediation action deployment
- Recorder configuration changes
- SSM Automation execution on production resources

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-remediation-configurations` | LOW | Remediation configuration |
| `describe-remediation-execution-status` | MEDIUM | Execution details |
| `get-automation-execution` | MEDIUM | SSM execution details |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules
- NEVER deploy untested remediation actions to production

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| IAM role fix | Revert IAM policy to previous version |
| SSM parameter fix | Revert remediation configuration |
| Remediation execution | Revert resource changes made by SSM automation |

## Output Format

```yaml
root_cause: "auto_remediation_failure — <specific_cause>"
evidence:
  - type: remediation_config
    content: "<describe-remediation-configurations output>"
  - type: remediation_execution
    content: "<remediation execution status>"
  - type: ssm_execution
    content: "<SSM Automation execution details>"
severity: HIGH
mitigation:
  immediate: "Fix the remediation role permissions or document parameters"
  long_term: "Test remediation documents and monitor execution status"
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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
