---
title: "D1 — RI/SP Recommendations"
description: "Diagnose issues with Reserved Instance and Savings Plans recommendations"
status: active
severity: MEDIUM
triggers:
  - "RI recommendation wrong"
  - "savings plan recommendation"
  - "reservation recommendation"
  - "commitment recommendation"
owner: devops-agent
objective: "Evaluate and troubleshoot RI/SP recommendation accuracy"
context: "Cost Explorer generates RI and SP recommendations based on historical usage. Recommendations assume consistent future usage. They consider lookback period (7, 30, or 60 days), payment option, and term length."
---

## Phase 1 — Triage

MUST:
- Get RI recommendations: `aws ce get-reservation-purchase-recommendation --service <service> --lookback-period-in-days SIXTY_DAYS --term-in-years ONE_YEAR --payment-option NO_UPFRONT`
- Get SP recommendations: `aws ce get-savings-plans-purchase-recommendation --savings-plans-type COMPUTE_SP --lookback-period-in-days SIXTY_DAYS --term-in-years ONE_YEAR --payment-option NO_UPFRONT`
- Review current usage patterns for the recommended services
- Check if usage has been stable during the lookback period

SHOULD:
- Compare recommendations across different lookback periods
- Check current RI/SP inventory: `aws ce get-reservation-utilization --time-period Start=<start>,End=<end>`
- Verify recommendations account for existing commitments

MAY:
- Analyze usage variability to assess recommendation risk
- Check if planned changes would affect recommendation validity

## Phase 2 — Remediate

MUST:
- Validate recommendations against actual usage patterns
- Consider usage stability before committing
- Start with shorter terms (1 year) if usage is uncertain

SHOULD:
- Use Compute Savings Plans for flexibility across instance types
- Purchase incrementally rather than all at once
- Monitor utilization after purchase

MAY:
- Use RI Marketplace to sell unused RIs
- Implement automated RI/SP management tools

## Common Issues

- symptoms: "Recommendations seem too aggressive"
  diagnosis: "Short lookback period captured peak usage, not average."
  resolution: "Use 60-day lookback for more stable recommendations."

- symptoms: "No recommendations generated"
  diagnosis: "Insufficient usage history or all usage already covered."
  resolution: "Check if account has enough history. Review existing RI/SP coverage."

- symptoms: "Recommendations don't match expected savings"
  diagnosis: "Recommendations based on on-demand pricing, not current blended rate."
  resolution: "Compare against actual on-demand cost, not current blended cost."

## Output Format

```yaml
root_cause: "ri_sp_recommendations — <specific_cause>"
evidence:
  - type: recommendations
    content: "<RI/SP recommendation details>"
  - type: usage_pattern
    content: "<historical usage analysis>"
severity: MEDIUM
mitigation:
  immediate: "Validate recommendations against usage patterns"
  long_term: "Implement regular RI/SP review and optimization cycle"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — RI/SP recommendations reveal usage patterns and financial commitments |
| Blast Radius | HIGH — Purchasing RIs/SPs creates long-term financial commitments |
| Rollback Complexity | HIGH — RI/SP purchases cannot be easily reversed (1-3 year terms) |
| Compliance Scope | HIGH — Financial commitments require procurement approval |

## Escalation Conditions

- Escalate to finance BEFORE purchasing any RIs or Savings Plans
- Escalate to management if recommendations suggest commitments exceeding budget authority
- Escalate to operations if usage patterns are expected to change significantly

## Data Sensitivity

- **Recommendation details** reveal usage patterns and potential savings — financial data
- **Purchase decisions** are long-term financial commitments — require proper authorization

## PROHIBITED Actions

- Do NOT purchase RIs or Savings Plans without finance team approval
- Do NOT commit to 3-year terms without thorough usage stability analysis
- Do NOT purchase based on short lookback periods (7 days) for variable workloads
- Do NOT purchase all recommended capacity at once — use incremental purchasing

## Phase 3 — Rollback

- Standard RIs can be listed on the RI Marketplace for resale
- Convertible RIs can be exchanged for different configurations
- Savings Plans cannot be cancelled — adjust future purchasing strategy
- Document lessons learned for future purchasing decisions

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
