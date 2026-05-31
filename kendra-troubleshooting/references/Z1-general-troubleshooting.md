---
title: "Z1 — General Kendra Troubleshooting"
description: "Catch-all for Kendra issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Kendra error"
  - "something wrong with Kendra"
  - "Kendra not working"
owner: devops-agent
objective: "Systematic triage for unclassified Kendra issues"
context: "Use when symptoms don't clearly map to a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check service health: AWS Health Dashboard
- List indices: `aws kendra list-indices`
- Check IAM permissions
- Review CloudTrail for recent errors

SHOULD:
- Check service quotas
- Verify region availability
- Review recent configuration changes

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B3, C1-C3, D1-D2, E1-E2, F1-F2, or G1-G2.

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: service_status
    content: "<Kendra service health>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Improve monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete indexes that serve production search applications
- Never modify data source connectors during active sync operations
- Never remove FAQ entries without confirming impact on search quality
- Never alter access control configurations without security review
- Never delete experience builder applications used by end users

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous data source connector configuration if sync operations fail
- Revert index relevance tuning settings if search quality degrades
- Re-enable previous ACL configurations if access control breaks

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
