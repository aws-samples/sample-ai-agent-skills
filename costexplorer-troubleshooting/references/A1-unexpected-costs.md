---
title: "A1 — Unexpected Costs"
description: "Diagnose unexpected cost increases or spikes"
status: active
severity: HIGH
triggers:
  - "unexpected cost"
  - "cost spike"
  - "bill higher than expected"
  - "surprise charges"
owner: devops-agent
objective: "Identify the root cause of unexpected cost increases"
context: "Unexpected costs are caused by unintended resource provisioning, data transfer charges, forgotten resources, auto-scaling events, or pricing changes. Cost Explorer data has a 24-48 hour delay."
---

## Phase 1 — Triage

MUST:
- Get cost breakdown by service: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity DAILY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE`
- Compare with previous period: `aws ce get-cost-and-usage --time-period Start=<prev-start>,End=<prev-end> --granularity DAILY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE`
- Identify top cost-increasing service
- Drill into the top service by usage type: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity DAILY --metrics UnblendedCost UsageQuantity --group-by Type=DIMENSION,Key=USAGE_TYPE --filter '{"Dimensions":{"Key":"SERVICE","Values":["<service>"]}}'`

SHOULD:
- Check by region: `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity DAILY --metrics UnblendedCost --group-by Type=DIMENSION,Key=REGION`
- Check by linked account (if Organizations): `aws ce get-cost-and-usage --time-period Start=<start>,End=<end> --granularity DAILY --metrics UnblendedCost --group-by Type=DIMENSION,Key=LINKED_ACCOUNT`
- Check for anomalies: `aws ce get-anomalies --date-interval StartDate=<start>,EndDate=<end>`

MAY:
- Check data transfer costs specifically
- Review marketplace charges
- Check for new resource types not previously used

## Phase 2 — Remediate

MUST:
- Identify and terminate/stop unneeded resources causing the cost increase
- Set up budget alerts to catch future spikes early
- Document the root cause for future reference

SHOULD:
- Create cost anomaly monitors for the affected services
- Implement tagging strategy for cost attribution
- Set up daily cost reports

MAY:
- Implement AWS Config rules to detect non-compliant resource creation
- Set up budget actions to automatically restrict spending

## Common Issues

- symptoms: "Bill doubled compared to last month"
  diagnosis: "New resources provisioned, auto-scaling event, or data transfer spike."
  resolution: "Drill into service and usage type. Identify new or increased usage."

- symptoms: "Charges from unknown service"
  diagnosis: "Marketplace subscription, free tier expiration, or cross-region resources."
  resolution: "Check marketplace subscriptions. Review free tier usage. Check all regions."

- symptoms: "Data transfer costs unexpectedly high"
  diagnosis: "Cross-region or internet-bound data transfer charges."
  resolution: "Identify data transfer sources. Use VPC endpoints or CloudFront to reduce costs."

## Output Format

```yaml
root_cause: "unexpected_costs — <specific_cause>"
evidence:
  - type: cost_breakdown
    content: "<service and usage type cost analysis>"
  - type: comparison
    content: "<period-over-period comparison>"
severity: HIGH
mitigation:
  immediate: "Terminate unneeded resources and set up budget alerts"
  long_term: "Implement cost monitoring, tagging, and anomaly detection"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Cost data reveals spending patterns and organizational structure |
| Blast Radius | LOW — Read-only cost analysis; no infrastructure changes |
| Rollback Complexity | LOW — No changes to roll back from analysis |
| Compliance Scope | MEDIUM — Cost data may be subject to financial reporting controls |

## Escalation Conditions

- Escalate IMMEDIATELY to management if unexpected costs exceed budget thresholds
- Escalate to security if cost spikes indicate unauthorized resource provisioning (potential compromise)
- Escalate to finance if cost anomalies affect financial reporting
- Escalate to AWS Support for billing disputes or unexplained charges

## Data Sensitivity

- **Cost data** reveals organizational spending patterns — restrict access to authorized personnel
- **Linked account costs** may expose business unit budgets — handle per financial data policies
- **Service-level costs** can reveal technology stack details — treat as confidential

## PROHIBITED Actions

- Do NOT share detailed cost breakdowns outside authorized finance and operations teams
- Do NOT terminate resources to reduce costs without confirming they are not in use
- Do NOT ignore cost spikes — they may indicate security incidents
- Do NOT modify billing settings without finance team approval

## Phase 3 — Rollback

- No rollback needed for cost analysis operations
- If resources were terminated to reduce costs, restore from backups if needed
- If budget alerts were misconfigured during troubleshooting, restore previous alert settings
- Document the cost investigation findings for financial records

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
