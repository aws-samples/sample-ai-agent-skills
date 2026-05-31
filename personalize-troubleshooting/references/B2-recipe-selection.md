---
title: "B2 — Recipe Selection Issues"
description: "Diagnose recipe selection and compatibility problems"
status: active
severity: MEDIUM
triggers: ["wrong recipe", "recipe not compatible", "recommendation type wrong"]
owner: devops-agent
objective: "Identify and resolve recipe selection issues"
context: "Recipes define the algorithm: USER_PERSONALIZATION for user recs, RELATED_ITEMS for similar items, PERSONALIZED_RANKING for re-ranking. Recipe must match use case."
---

## Phase 1 — Triage

MUST:
- Check solution recipe: `aws personalize describe-solution --solution-arn <arn>`
- Verify recipe matches use case
- List available recipes: `aws personalize list-recipes`
- Check recipe requirements vs available data

SHOULD:
- Verify recipe supports available dataset types
- Check recipe-specific hyperparameters
- Compare recipe capabilities

MAY:
- Test multiple recipes for quality comparison
- Review recipe documentation

## Phase 2 — Remediate

MUST:
- Select recipe matching use case (USER_PERSONALIZATION, RELATED_ITEMS, PERSONALIZED_RANKING)
- Ensure required datasets exist for chosen recipe
- Create new solution with correct recipe (cannot change recipe on existing solution)

SHOULD:
- Test with default hyperparameters first
- Evaluate recommendation quality with metrics

MAY:
- Implement A/B testing across recipes

## Common Issues

- symptoms: "GetRecommendations returns items instead of user recommendations"
  diagnosis: "Using RELATED_ITEMS recipe instead of USER_PERSONALIZATION."
  resolution: "Create new solution with USER_PERSONALIZATION recipe."

- symptoms: "Recipe requires dataset that doesn't exist"
  diagnosis: "Recipe needs Users or Items dataset not yet created."
  resolution: "Create and import required datasets before training."

## Output Format

```yaml
root_cause: "recipe_selection — <specific_cause>"
evidence:
  - type: solution_config
    content: "<solution and recipe details>"
severity: MEDIUM
mitigation:
  immediate: "Create solution with correct recipe"
  long_term: "Document recipe selection criteria"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — recipe selection is a configuration decision
- Service Disruption Risk: LOW — recipe issues affect model quality, not availability
- Compliance Impact: LOW — recipe selection is a development concern

## Escalation Conditions
- Selected recipe incompatible with dataset schema
- Recipe produces poor recommendation quality
- Recipe requires data fields not available in dataset
- Recipe training costs exceed expectations
- Recipe deprecated or no longer supported

## Data Sensitivity
- Recipe configurations — INTERNAL
- Dataset compatibility details — INTERNAL
- Performance benchmarks — INTERNAL
- Recipe selection rationale — INTERNAL
- Never expose proprietary recommendation strategies

## Prohibited Actions
- Do NOT select recipes without understanding data requirements
- Do NOT use deprecated recipes for new solutions
- Do NOT ignore recipe compatibility warnings
- Do NOT bypass recipe evaluation before production deployment
- Do NOT select recipes based solely on cost without quality consideration
- Do NOT change recipes on production solutions without testing

## Phase 3 — Rollback
- Revert to previous recipe selection
- Restore solution configuration with previous recipe
- Re-train with the original recipe
- Verify recommendation quality after recipe change
- Document recipe selection rationale and evaluation results
- Update recipe selection guidelines

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
