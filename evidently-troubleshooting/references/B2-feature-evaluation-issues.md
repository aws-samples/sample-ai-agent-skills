---
title: "B2 — Feature Evaluation Issues"
description: "Diagnose feature evaluation failures and unexpected results"
status: active
severity: HIGH
triggers: ["wrong variation returned", "evaluation failed", "EvaluateFeature error", "unexpected feature value"]
owner: devops-agent
objective: "Resolve feature evaluation issues including wrong variations and evaluation failures"
context: "Feature evaluation returns a variation based on active launches, experiments, overrides, or the default variation. Evaluation strategy (client-side vs server-side) affects behavior."
---

## Phase 1 — Triage

MUST:
- Test evaluation: `aws evidently evaluate-feature --project <name> --feature <feature-name> --entity-id <entity-id>`
- Get feature details: `aws evidently get-feature --project <name> --feature <feature-name>`
- Check for active launches and experiments on this feature

SHOULD:
- Verify entity overrides: check if the entity has a specific override mapping
- Check the evaluation strategy (client-side vs server-side)
- Review active launch traffic splits and experiment allocations

MAY:
- Test with multiple entity IDs to verify traffic distribution
- Check CloudTrail for EvaluateFeature errors
- Verify the SDK version if using client-side evaluation

## Phase 2 — Remediate

MUST:
- If wrong variation: check override rules, launch rules, experiment rules, then default variation (in precedence order)
- If evaluation fails: verify IAM permissions for evidently:EvaluateFeature
- Ensure the feature exists and the project is AVAILABLE

SHOULD:
- Remove incorrect entity overrides if they cause wrong evaluations
- Verify launch traffic splits match expected percentages
- Check experiment allocation if the entity is in an experiment

MAY:
- Update the default variation if it is incorrect
- Adjust launch or experiment traffic allocation

## Common Issues

- symptoms: "Entity always gets the same variation regardless of launch configuration"
  diagnosis: "Entity may have an override that takes precedence over launch rules."
  resolution: "Check entity overrides: `aws evidently get-feature --project <name> --feature <feature-name>`. Remove override if unintended: `aws evidently update-feature --project <name> --feature <feature-name> --remove-entity-overrides <entity-id>`"

## Safety Ratings

```
safety_ratings:
  - "evaluate-feature: GREEN — read-only evaluation test"
  - "get-feature: GREEN — read-only feature details"
  - "update-feature remove-entity-overrides: YELLOW — changes evaluation behavior for specific entities"
  - "list-launches: GREEN — read-only launch enumeration"
```

## Escalation Conditions

- Evaluations return wrong variations for a large percentage of entities
- Evaluation latency exceeds acceptable thresholds
- Client-side evaluation produces different results than server-side
- Feature evaluation failures affecting production traffic

## Data Sensitivity

- `aws evidently evaluate-feature`: LOW — returns variation for a test entity
- `aws evidently get-feature`: LOW — returns feature configuration
- Entity overrides: MEDIUM — may reveal specific user-to-variation mappings

## Prohibited Actions

- NEVER modify feature variations during an active experiment
- NEVER remove entity overrides without understanding why they were set
- NEVER change the default variation on a production feature without testing
- NEVER assume client-side and server-side evaluation produce identical results without verification

## Phase 3 — Rollback

1. Restore entity overrides if they were removed: `aws evidently update-feature --project <name> --feature <feature-name> --add-or-update-entity-overrides <entity-id>=<variation>`
2. Revert default variation changes if made
3. Verify evaluations return expected results after changes
4. Document the evaluation issue and resolution
5. Confirm active launches and experiments are unaffected

## Output Format

```yaml
root_cause: "feature_evaluation — <specific_cause>"
evidence:
  - type: evaluation_issue
    content: "<evaluation details and expected vs actual variation>"
severity: HIGH
mitigation:
  immediate: "Fix evaluation rules or overrides"
  long_term: "Implement evaluation testing in CI/CD pipeline"
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
