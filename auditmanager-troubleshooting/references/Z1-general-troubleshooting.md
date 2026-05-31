---
title: "Z1 — General Audit Manager Troubleshooting"
description: "Catch-all runbook for Audit Manager issues"
status: active
severity: MEDIUM
triggers: ["Audit Manager error", "audit issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Audit Manager issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check settings: `aws auditmanager get-settings --attribute ALL`
- List assessments: `aws auditmanager list-assessments`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=auditmanager.amazonaws.com --max-results 20`

SHOULD:
- Verify AWS Config is enabled
- Check IAM permissions

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
root_cause: "general_auditmanager — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Identify problem domain and pivot to specific runbook | GREEN | Diagnostic routing — no system changes |
| Document findings | GREEN | Documentation only |
| Set up monitoring | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Unclassified issues may indicate systemic compliance problems
- Investigation may reveal sensitive compliance gaps

## Data Sensitivity

- General investigation may expose compliance posture across multiple assessments
- CloudTrail analysis may reveal sensitive API call patterns
- Settings review may expose account-wide audit configuration

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER disable Audit Manager during investigation
- NEVER modify active assessments during troubleshooting without documentation

## Phase 3 — Rollback

- General troubleshooting is diagnostic — rollback depends on the specific runbook identified
- If any settings were changed during investigation: restore original values
- Document all investigative actions for compliance audit trail

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
