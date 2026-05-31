---
title: "Z1 — General Control Catalog Troubleshooting"
description: "Catch-all runbook for Control Catalog issues"
status: active
severity: MEDIUM
triggers: ["Control Catalog error", "control catalog not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified Control Catalog issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Test API access: `aws controlcatalog list-controls --max-results 5`
- Check IAM permissions
- Verify CLI version supports controlcatalog
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=controlcatalog.amazonaws.com --max-results 20`

SHOULD:
- Check service availability in region
- Verify account has access to Control Catalog

MAY:
- Check AWS Health Dashboard

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings

MAY:
- Set up monitoring

## Output Format

```yaml
root_cause: "general_controlcatalog — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — General troubleshooting involves public catalog data |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | MEDIUM — Troubleshooting supports compliance program operations |

## Escalation Conditions

- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to AWS Support if systematic API failures affect all Control Catalog operations
- Escalate to compliance team if troubleshooting delays compliance assessments

## Data Sensitivity

- **CloudTrail events** may contain account identifiers — standard handling
- **API responses** contain public reference data

## PROHIBITED Actions

- Do NOT modify IAM permissions or account configurations during initial triage
- Do NOT share account-specific CloudTrail logs outside the operations team

## Phase 3 — Rollback

- No rollback needed for diagnostic operations
- Revert any configuration changes made during troubleshooting
- Route to the appropriate specific runbook for remediation

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
