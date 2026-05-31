---
title: "F2 — Consolidated Billing"
description: "Diagnose consolidated billing discrepancies and issues"
status: active
severity: MEDIUM
triggers:
  - "consolidated billing issue"
  - "billing discrepancy"
  - "organization billing"
  - "blended rate wrong"
owner: devops-agent
objective: "Resolve consolidated billing discrepancies in Organizations"
context: "Consolidated billing aggregates costs across all member accounts. RI and SP benefits are shared across the organization by default. Blended rates average discounts across accounts. Credits and support charges are applied at the management account level."
---

## Phase 1 — Triage

MUST:
- Get organizational cost breakdown: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity MONTHLY --metrics UnblendedCost BlendedCost --group-by Type=DIMENSION,Key=LINKED_ACCOUNT`
- Compare unblended vs blended costs for discrepancies
- Check RI/SP sharing settings: `aws organizations describe-organization`
- Verify credits and refunds: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=RECORD_TYPE`

SHOULD:
- Check if RI/SP sharing is enabled or disabled
- Review support plan charges across accounts
- Verify tax calculations for different account regions

MAY:
- Analyze blended rate calculations for specific services
- Check for marketplace charges across accounts

## Phase 2 — Remediate

MUST:
- Understand the difference between unblended, blended, and amortized costs
- Use unblended cost for actual per-account charges
- Use amortized cost for RI/SP analysis

SHOULD:
- Enable RI/SP sharing for maximum organizational savings
- Set up per-account budgets using unblended cost
- Document billing allocation methodology for stakeholders

MAY:
- Implement chargeback/showback using CUR data
- Create custom billing reports with cost categories

## Common Issues

- symptoms: "Account bill doesn't match Cost Explorer"
  diagnosis: "Using different cost metrics (blended vs unblended) or timing differences."
  resolution: "Use UnblendedCost for actual charges. Account for data delay."

- symptoms: "RI savings not appearing for member account"
  diagnosis: "RI sharing disabled or RI purchased in different account."
  resolution: "Enable RI sharing in Organizations. Check RI ownership account."

- symptoms: "Unexpected charges on management account"
  diagnosis: "Support plan, marketplace, or tax charges consolidated to management account."
  resolution: "Review RECORD_TYPE breakdown. Support and tax are management account charges."

## Output Format

```yaml
root_cause: "consolidated_billing — <specific_cause>"
evidence:
  - type: cost_breakdown
    content: "<per-account cost comparison>"
  - type: billing_config
    content: "<RI/SP sharing and organization settings>"
severity: MEDIUM
mitigation:
  immediate: "Clarify cost metric usage and fix sharing settings"
  long_term: "Implement chargeback model and billing documentation"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Consolidated billing reveals organizational financial structure |
| Blast Radius | MEDIUM — RI/SP sharing changes affect cost allocation across all accounts |
| Rollback Complexity | MEDIUM — Sharing settings can be changed but affect billing retroactively |
| Compliance Scope | HIGH — Consolidated billing affects financial reporting and chargeback |

## Escalation Conditions

- Escalate to finance if billing discrepancies affect financial reporting
- Escalate to management if RI/SP sharing changes affect account-level budgets
- Escalate to AWS Support for billing calculation disputes

## Data Sensitivity

- **Per-account cost breakdowns** reveal business unit spending — highly sensitive
- **RI/SP sharing configuration** affects financial allocation across the organization
- **Blended vs unblended costs** have different implications for chargeback

## PROHIBITED Actions

- Do NOT change RI/SP sharing settings without finance team approval
- Do NOT present blended costs as actual per-account charges
- Do NOT modify organizational billing settings without understanding cross-account impact
- Do NOT share per-account cost data outside authorized finance teams

## Phase 3 — Rollback

- Restore RI/SP sharing settings to previous configuration
- Recalculate chargeback/showback reports if sharing settings were changed
- Notify affected account owners of any billing allocation changes
- Document the change for financial audit trail

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
