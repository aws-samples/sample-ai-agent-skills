---
title: "Z1 — General Inspector Troubleshooting"
description: "Catch-all systematic investigation for Inspector issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Inspector issue"
  - "Inspector problem"
  - "scanning not working"
  - "Inspector error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Inspector issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Inspector issues that don't clearly match a specific runbook. It covers account status, coverage, findings, and integration checks. Use this as a starting point, then pivot to a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check account status: `aws inspector2 batch-get-account-status`
- Check coverage statistics: `aws inspector2 list-coverage-statistics`
- Check active findings summary: `aws inspector2 list-finding-aggregations --aggregation-type SEVERITY`
- Check delegated admin (if multi-account): `aws inspector2 get-delegated-admin-account`

SHOULD:
- Check for uncovered resources: `aws inspector2 list-coverage --filter-criteria '{"scanStatusCode":[{"comparison":"EQUALS","value":"INACTIVE"}]}'`
- Review suppression rules: `aws inspector2 list-filters`
- Check CloudTrail for Inspector events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=inspector2.amazonaws.com --max-results 20`

MAY:
- Check Security Hub integration status
- Review EventBridge rules for Inspector
- Check AWS Health Dashboard for service issues

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Scanning issues → A1 (EC2), A2 (ECR), A3 (Lambda)
  - Finding issues → B1 (analysis), B2 (false positives), B3 (suppression)
  - Coverage issues → C1 (gaps), C2 (SSM agent)
  - Integration → D1 (Security Hub), D2 (EventBridge)
  - Multi-account → E1 (delegated admin), E2 (member accounts)
  - Configuration → F1 (scan frequency), F2 (exclusion rules)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent configuration changes

MAY:
- Set up comprehensive monitoring
- Create post-incident documentation

## Common Issues

- symptoms: "Inspector not working — unclear what's wrong"
  diagnosis: "Multiple possible causes. Follow systematic triage."
  resolution: "Check account status → coverage → findings → integrations."

- symptoms: "Inspector was working but stopped"
  diagnosis: "Service disabled, IAM changes, or SSM agent issues."
  resolution: "Check account status. Verify IAM roles. Check SSM agent on EC2."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Pivot to specific runbook | GREEN | Investigation only, no state changes |
| Check account status and coverage | GREEN | Read-only operations |
| Review recent changes | GREEN | Read-only analysis |

## Escalation Conditions

- Inspector not functioning and issue cannot be classified — escalate to AWS Support
- Multiple concurrent scanning issues — escalate to incident management
- Inspector disabled unexpectedly — escalate immediately to security team
- Organization-wide scanning issues — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status |
| `aws inspector2 list-coverage-statistics` | MEDIUM | Exposes coverage metrics |
| `aws inspector2 list-finding-aggregations` | MEDIUM | Exposes finding statistics |
| `aws inspector2 get-delegated-admin-account` | HIGH | Exposes delegated admin account |
| `aws cloudtrail lookup-events` (Inspector) | HIGH | Exposes API call history |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
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
  - type: account_status
    content: "<Inspector account status>"
  - type: coverage
    content: "<coverage statistics>"
  - type: findings
    content: "<finding summary>"
severity: MEDIUM
general_analysis:
  scanning_status: "enabled | disabled | partial"
  coverage: "full | gaps identified"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
