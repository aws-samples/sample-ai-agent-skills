---
title: "D1 — Differential Privacy Issues"
description: "Diagnose Clean Rooms differential privacy problems including budget management and configuration"
status: active
severity: MEDIUM
triggers:
  - "differential privacy"
  - "privacy budget"
  - "epsilon budget"
  - "dp configuration"
  - "privacy exhausted"
owner: devops-agent
objective: "Resolve differential privacy configuration and budget issues"
context: "Clean Rooms differential privacy adds mathematical noise to query results to protect individual records. It uses a finite privacy budget (epsilon) that depletes with each query. Common issues include budget exhaustion, configuration errors, and noise impact on results."
---

## Phase 1 — Triage

MUST:
- Check privacy budget template: `aws cleanrooms list-privacy-budget-templates --membership-identifier <membership-id>`
- Get budget details: `aws cleanrooms get-collaboration-privacy-budget-template --collaboration-identifier <collab-id> --privacy-budget-template-identifier <template-id>`
- Check remaining budget: `aws cleanrooms list-privacy-budgets --membership-identifier <membership-id>`
- Verify differential privacy is enabled on the collaboration

SHOULD:
- Check budget consumption rate
- Verify the privacy budget refresh schedule
- Review query patterns that consume the most budget

MAY:
- Check CloudTrail for privacy budget events
- Compare budget consumption across different query types

## Phase 2 — Remediate

MUST:
- If budget exhausted, wait for refresh or create new budget template
- If not configured, create privacy budget template:
  ```
  aws cleanrooms create-privacy-budget-template --membership-identifier <membership-id> --auto-refresh CALENDAR_MONTH --privacy-budget-type DIFFERENTIAL_PRIVACY --parameters '{"differentialPrivacy":{"epsilon":10,"usersNoisePerQuery":30}}'
  ```
- Adjust epsilon value based on privacy vs. utility requirements

SHOULD:
- Monitor budget consumption and set alerts
- Optimize queries to minimize budget consumption

## Output Format

```yaml
root_cause: "differential_privacy — <specific_cause>"
evidence:
  - type: budget_status
    content: "<privacy budget details>"
  - type: budget_template
    content: "<privacy budget template>"
severity: MEDIUM
mitigation:
  immediate: "Wait for budget refresh or adjust configuration"
  long_term: "Monitor budget consumption and optimize query patterns"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Wait for budget refresh | GREEN |
| Create privacy budget template | RED |
| Adjust epsilon value | RED |
| Monitor budget consumption | GREEN |
| Optimize queries to minimize budget consumption | GREEN |

## Escalation Conditions

- Fix requires modifying differential privacy parameters (epsilon, noise)
- Fix involves increasing privacy budget beyond established thresholds

## Data Sensitivity

- **CRITICAL**: differential privacy parameters directly control the mathematical privacy guarantees
- Epsilon values and noise configuration determine the level of individual record protection

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If privacy budget template was created with wrong parameters, delete and recreate with correct epsilon
2. If epsilon was increased, restore previous lower value to maintain privacy guarantees
3. If auto-refresh was changed, restore previous refresh schedule
4. Verify rollback by confirming `aws cleanrooms list-privacy-budgets` shows expected privacy configuration

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
