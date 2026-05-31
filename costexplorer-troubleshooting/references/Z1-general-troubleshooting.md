---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Cost Explorer issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "cost explorer problem"
  - "billing issue"
  - "budget problem"
  - "cost management issue"
owner: devops-agent
objective: "Systematically diagnose unspecified Cost Explorer or Budgets issues"
context: "This catch-all runbook provides a systematic approach when the specific issue category is unclear. It covers initial data collection, common checks, and routing to specific runbooks."
---

## Phase 1 — Triage

MUST:
- Check overall cost trend: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE`
- Check budget status: `aws budgets describe-budgets --account-id <account-id>`
- Check for anomalies: `aws ce get-anomalies --date-interval StartDate=<start>,EndDate=<end>`
- Verify Cost Explorer access and permissions

SHOULD:
- Check account type (management vs member in Organizations)
- Review recent cost changes by service
- Check CUR report status: `aws cur describe-report-definitions --region us-east-1`

MAY:
- Check AWS Health Dashboard for billing service events
- Review IAM permissions for cost management

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If cost analysis issue → A1-A3
- If budget issue → B1-B3
- If anomaly issue → C1-C2
- If savings issue → D1-D2
- If reporting issue → E1-E2
- If organization issue → F1-F2

SHOULD:
- Document findings and route to specific runbook
- Collect all relevant evidence before escalating

MAY:
- Contact AWS Support for billing-specific issues
- Review AWS billing documentation for recent changes

## Common Issues

- symptoms: "Cost Explorer not working (unspecified)"
  diagnosis: "Multiple possible causes. Systematic triage required."
  resolution: "Follow Phase 1 triage to identify category, then route to specific runbook."

- symptoms: "Cannot access Cost Explorer"
  diagnosis: "IAM permissions missing or Cost Explorer not enabled."
  resolution: "Grant ce:* permissions. Enable Cost Explorer in Billing console."

- symptoms: "Cost data seems wrong"
  diagnosis: "Using wrong cost metric, data delay, or organizational billing confusion."
  resolution: "Clarify cost metric. Account for 24-48 hour delay. Check account scope."

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: cost_overview
    content: "<cost trend and budget status>"
  - type: triage_findings
    content: "<initial diagnostic results>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook based on triage"
  long_term: "Implement comprehensive cost monitoring and governance"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — General troubleshooting may expose cost and billing data |
| Blast Radius | LOW — Diagnostic commands are read-only |
| Rollback Complexity | LOW — No changes during triage |
| Compliance Scope | MEDIUM — Cost management supports financial governance |

## Escalation Conditions

- Escalate to the specific runbook owner once the problem domain is identified
- Escalate to finance if cost management tools are non-functional
- Escalate to AWS Support for billing-specific issues

## Data Sensitivity

- **Cost data** accessed during troubleshooting is financial information — restrict access
- **CloudTrail events** may contain account identifiers

## PROHIBITED Actions

- Do NOT modify billing settings during initial triage
- Do NOT share cost data outside authorized teams
- Do NOT make financial decisions based on incomplete troubleshooting data

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
