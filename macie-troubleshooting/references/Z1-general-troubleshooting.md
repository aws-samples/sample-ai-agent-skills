---
title: "Z1 — General Macie Troubleshooting"
description: "Catch-all systematic investigation for Macie issues"
status: active
severity: MEDIUM
triggers:
  - "Macie issue"
  - "Macie problem"
  - "Macie not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Macie issues using a structured diagnostic approach"
context: "General-purpose investigation framework for Macie issues. Covers session status, job health, findings, and integrations. Use as starting point, then pivot to specific runbook."
---

## Phase 1 — Triage

MUST:
- Check Macie session: `aws macie2 get-macie-session`
- List jobs: `aws macie2 list-classification-jobs --sort-criteria '{"attributeName":"createdAt","orderBy":"DESC"}' --max-results 10`
- Check findings summary: `aws macie2 get-finding-statistics --finding-criteria '{}' --group-by '{"key":"severity.description"}'`
- Check usage: `aws macie2 get-usage-totals`

SHOULD:
- Check bucket inventory: `aws macie2 describe-buckets --max-results 10`
- Review custom identifiers: `aws macie2 list-custom-data-identifiers`
- Check multi-account setup: `aws macie2 get-administrator-account`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=macie2.amazonaws.com --max-results 20`

MAY:
- Check Security Hub integration
- Review EventBridge rules
- Check AWS Health Dashboard

## Phase 2 — Remediate

MUST:
- Pivot to specific runbook based on findings:
  - Discovery → A1 (jobs), A2 (classification), A3 (custom identifiers)
  - Findings → B1 (analysis), B2 (false positives)
  - S3 → C1 (inventory), C2 (discovery)
  - Multi-Account → D1 (admin), D2 (members)
  - Cost → E1 (optimization), E2 (sampling)
  - Integration → F1 (Security Hub), F2 (EventBridge)

SHOULD:
- Check for multiple concurrent issues
- Review recent configuration changes

MAY:
- Set up comprehensive monitoring
- Create documentation for the issue

## Common Issues

- symptoms: "Macie not working — unclear what's wrong"
  diagnosis: "Follow systematic triage."
  resolution: "Check session → jobs → findings → integrations."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Pivot to specific runbook | GREEN | Investigation only, no state changes |
| Check session and job status | GREEN | Read-only operations |
| Review recent changes | GREEN | Read-only analysis |

## Escalation Conditions

- Macie not functioning and issue cannot be classified — escalate to AWS Support
- Multiple concurrent issues — escalate to incident management
- Macie disabled unexpectedly — escalate immediately to security team
- Organization-wide Macie issues — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 get-macie-session` | LOW | Session status only |
| `aws macie2 list-classification-jobs` | HIGH | Exposes job configurations |
| `aws macie2 get-finding-statistics` | MEDIUM | Exposes finding counts |
| `aws macie2 get-usage-totals` | MEDIUM | Exposes usage and cost |
| `aws cloudtrail lookup-events` (Macie) | HIGH | Exposes API call history |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** make state-changing remediations during general troubleshooting without first classifying the issue

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| General triage is read-only | No rollback needed for Phase 1 triage |
| Pivot to specific runbook | Follow rollback steps in the specific runbook |

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: session_status
    content: "<Macie session status>"
  - type: jobs
    content: "<job status summary>"
  - type: findings
    content: "<finding summary>"
severity: MEDIUM
general_analysis:
  macie_status: "enabled | disabled"
  jobs: "running | failed | none"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow identified specific runbook"
  long_term: "Implement comprehensive monitoring"
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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
