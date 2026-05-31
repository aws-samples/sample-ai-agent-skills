---
title: "C2 — Privacy Budget Issues"
description: "Diagnose privacy budget exhaustion, configuration, and consumption problems"
status: active
severity: CRITICAL
triggers: ["privacy budget exhausted", "budget exceeded", "no budget remaining", "budget error"]
owner: devops-agent
objective: "Resolve privacy budget issues and restore audience generation capability"
context: "Privacy budgets limit the number of audience generation jobs to enforce differential privacy. Budget consumption is irreversible. Exhausted budgets block all audience generation until refresh."
---

## Phase 1 — Triage

MUST:
- Check configured audience model for budget settings: `aws cleanroomsml get-configured-audience-model --configured-audience-model-arn <arn>`
- List recent audience generation jobs to assess consumption: `aws cleanroomsml list-audience-generation-jobs --configured-audience-model-arn <arn>`
- Verify budget refresh schedule and remaining capacity

SHOULD:
- Calculate total budget consumed vs. allocated
- Check if failed jobs consumed budget (they may or may not depending on failure point)

MAY:
- Review historical job patterns to predict budget exhaustion timeline
- Check CloudTrail for budget-related API calls

## Phase 2 — Remediate

MUST:
- Wait for budget period refresh if budget is exhausted — there is no manual reset
- Plan future audience generation jobs to stay within budget limits
- Verify budget allocation is sufficient for planned job frequency

SHOULD:
- Implement budget monitoring alerts before exhaustion
- Coordinate with collaboration members on budget allocation changes

MAY:
- Request budget increase through collaboration configuration update (requires member agreement)
- Optimize audience generation frequency to conserve budget

## Common Issues

- symptoms: "Audience generation job rejected — privacy budget exhausted"
  diagnosis: "All allocated privacy budget has been consumed for the current period."
  resolution: "Wait for the budget period to refresh. Plan jobs to stay within budget. Request budget increase if needed."

- symptoms: "Budget consumed faster than expected"
  diagnosis: "Failed jobs or test jobs consumed budget without producing useful output."
  resolution: "Implement pre-flight validation before starting jobs. Track budget consumption per job."

## Safety Ratings

```
safety_ratings:
  - "get-configured-audience-model: GREEN — read-only budget information"
  - "list-audience-generation-jobs: GREEN — read-only job history"
  - "update-configured-audience-model: YELLOW — may change budget parameters"
  - "start-audience-generation-job: RED — irreversibly consumes privacy budget"
```

## Escalation Conditions

- Privacy budget exhausted with critical business audience generation pending
- Budget refresh period is weeks away and business cannot wait
- Collaboration members disagree on budget allocation changes
- Suspected budget consumption from unauthorized audience generation jobs

## Data Sensitivity

- `aws cleanroomsml get-configured-audience-model`: MEDIUM — reveals budget configuration
- `aws cleanroomsml list-audience-generation-jobs`: MEDIUM — reveals job history and consumption
- Budget allocation details: MEDIUM — reveals collaboration privacy parameters
- Budget consumption patterns: LOW — operational metric

## Prohibited Actions

- NEVER attempt to reset or reverse privacy budget consumption — it is mathematically irreversible
- NEVER start audience generation jobs without verifying remaining budget
- NEVER change budget parameters without agreement from all collaboration members
- NEVER suggest workarounds that circumvent privacy budget enforcement

## Phase 3 — Rollback

1. Privacy budget consumption cannot be rolled back — it is by design irreversible
2. If budget parameters were changed incorrectly, update the configured audience model with correct values
3. If unauthorized jobs consumed budget, document the incident and review access controls
4. Implement budget monitoring to prevent future exhaustion
5. Notify collaboration members of budget status and any parameter changes

## Output Format

```yaml
root_cause: "privacy_budget_issue — <specific_cause>"
evidence:
  - type: budget_error
    content: "<budget status and consumption details>"
severity: CRITICAL
mitigation:
  immediate: "Wait for budget refresh or coordinate budget increase"
  long_term: "Implement budget monitoring and job scheduling"
```

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
