---
title: "Z1 — General Comprehend Troubleshooting"
description: "Catch-all runbook for Comprehend issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "comprehend error"
  - "comprehend not working"
  - "NLP issue"
owner: devops-agent
objective: "Systematically diagnose unspecified Comprehend issues"
context: "Use this runbook when the issue doesn't clearly map to a specific runbook."
---

## Phase 1 — Triage

MUST:
- Identify which Comprehend API is failing
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=comprehend.amazonaws.com --max-results 20`
- Verify IAM permissions
- Check text input and language code

SHOULD:
- Test with simple DetectEntities call
- Check for custom models and endpoints
- Review service quotas

MAY:
- Check service health dashboard
- Review account-level limits

## Phase 2 — Remediate

MUST:
- Route to specific runbook once problem domain identified

SHOULD:
- Check multiple problem domains if initial investigation inconclusive

MAY:
- Engage AWS Support if issue persists

## Common Issues

- symptoms: "General Comprehend errors"
  diagnosis: "Start with API identification, then permissions, then input validation."
  resolution: "Follow systematic triage. Route to specific runbook."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: api_errors
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive Comprehend monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — General troubleshooting may involve examining various Comprehend resources |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes made during triage |
| Compliance Scope | MEDIUM — Ensure troubleshooting does not expose sensitive data |

## Escalation Conditions

- Escalate to security if troubleshooting reveals unauthorized Comprehend API usage
- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to AWS Support if systematic issues affect multiple Comprehend APIs

## Data Sensitivity

- **CloudTrail events** may reveal API call patterns but not document content
- **Troubleshooting artifacts** should not include raw text data from Comprehend inputs
- **IAM policy reviews** may expose permission configurations — handle appropriately

## PROHIBITED Actions

- Do NOT use production data for troubleshooting tests — use synthetic data
- Do NOT modify IAM permissions or service configurations during initial triage
- Do NOT share CloudTrail logs containing account details outside the operations team

## Phase 3 — Rollback

- No rollback needed for diagnostic operations
- If any configuration was changed during troubleshooting, document and revert
- Route to the appropriate specific runbook for remediation actions

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
