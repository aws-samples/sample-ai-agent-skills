---
title: "D2 — Utilization"
description: "Diagnose low RI/SP utilization issues"
status: active
severity: MEDIUM
triggers:
  - "low RI utilization"
  - "savings plan underutilized"
  - "unused reservations"
  - "wasted commitment"
owner: devops-agent
objective: "Identify and resolve low RI/SP utilization"
context: "Low utilization means you are paying for committed capacity that is not being used. RI utilization below 80% indicates potential waste. Causes include instance type changes, workload reduction, or incorrect RI scope."
---

## Phase 1 — Triage

MUST:
- Check RI utilization: `aws ce get-reservation-utilization --time-period Start=<start>,End=<end> --group-by Type=DIMENSION,Key=SUBSCRIPTION_ID`
- Check SP utilization: `aws ce get-savings-plans-utilization --time-period Start=<start>,End=<end>`
- Identify underutilized RIs by instance type and region
- Check SP utilization details: `aws ce get-savings-plans-utilization-details --time-period Start=<start>,End=<end>`

SHOULD:
- Check RI coverage gaps: `aws ce get-reservation-coverage --time-period Start=<start>,End=<end> --group-by Type=DIMENSION,Key=INSTANCE_TYPE`
- Review if instance types have changed since RI purchase
- Check if workloads have been decommissioned or migrated

MAY:
- Analyze utilization trends over time
- Check if RIs can be modified to match current usage

## Phase 2 — Remediate

MUST:
- Identify which RIs are underutilized and why
- Modify RIs to match current instance types (same family, same region)
- Right-size instances to match RI commitments

SHOULD:
- List unused RIs on RI Marketplace for sale
- Convert Standard RIs to Convertible if flexibility needed
- Adjust future purchasing strategy based on utilization data

MAY:
- Implement automated RI management and optimization
- Set up utilization monitoring with CloudWatch alarms

## Common Issues

- symptoms: "RI utilization below 50%"
  diagnosis: "Instance type changed or workload decommissioned after RI purchase."
  resolution: "Modify RI to match current instance type or sell on Marketplace."

- symptoms: "SP utilization declining over time"
  diagnosis: "Workload reduction or migration to non-covered services."
  resolution: "Review SP coverage. Compute SP covers EC2, Fargate, Lambda."

- symptoms: "RI shows 0% utilization"
  diagnosis: "No matching instances running in the RI's scope (region/AZ)."
  resolution: "Check RI scope. Modify to match current deployment region/AZ."

## Output Format

```yaml
root_cause: "utilization — <specific_cause>"
evidence:
  - type: ri_utilization
    content: "<RI utilization by subscription>"
  - type: sp_utilization
    content: "<SP utilization details>"
severity: MEDIUM
mitigation:
  immediate: "Modify RIs or right-size instances to improve utilization"
  long_term: "Implement utilization monitoring and regular optimization reviews"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Utilization data reveals commitment efficiency |
| Blast Radius | MEDIUM — RI modifications affect cost allocation across the organization |
| Rollback Complexity | MEDIUM — RI modifications can be reversed but may take time |
| Compliance Scope | MEDIUM — Utilization monitoring supports financial governance |

## Escalation Conditions

- Escalate to finance if RI/SP utilization drops below 50%
- Escalate to management if wasted commitment costs exceed thresholds
- Escalate to operations if instance type changes are needed to improve utilization

## Data Sensitivity

- **Utilization data** reveals financial efficiency — treat as financial data
- **RI subscription details** contain commitment terms and costs

## PROHIBITED Actions

- Do NOT modify RIs without understanding the impact on all accounts in the organization
- Do NOT sell RIs on the Marketplace without finance approval
- Do NOT change instance types solely to match RIs without considering application requirements

## Phase 3 — Rollback

- Reverse RI modifications if they negatively affect utilization
- Restore previous instance types if changes were made to match RIs
- Document utilization changes for financial reporting

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
