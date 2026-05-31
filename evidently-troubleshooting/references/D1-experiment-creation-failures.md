---
title: "D1 — Experiment Creation Failures"
description: "Diagnose experiment creation and configuration failures"
status: active
severity: HIGH
triggers: ["experiment creation failed", "CreateExperiment error", "cannot create experiment", "metric goal error"]
owner: devops-agent
objective: "Resolve experiment creation issues including metric goals and treatment configuration"
context: "Experiments measure the impact of feature variations using metrics and Bayesian inference. They require metric definitions, treatments, and a baseline variation."
---

## Phase 1 — Triage

MUST:
- List experiments in the project: `aws evidently list-experiments --project <name>`
- Verify the feature referenced by the experiment exists: `aws evidently get-feature --project <name> --feature <feature-name>`
- Check IAM permissions for evidently:CreateExperiment

SHOULD:
- Verify metric definitions are valid (metric name, entity ID key, value key, event pattern)
- Check that treatments reference valid feature variations
- Verify a baseline treatment is specified

MAY:
- Review CloudTrail for CreateExperiment errors
- Check if the project has data delivery configured for experiment data

## Phase 2 — Remediate

MUST:
- Fix metric goal definitions with valid metric names and event patterns
- Ensure treatments reference existing feature variations
- Set a valid baseline treatment

SHOULD:
- Configure appropriate sample size for statistical significance
- Set experiment duration based on expected traffic volume
- Configure data delivery at the project level for experiment data export

MAY:
- Define multiple metric goals for comprehensive analysis
- Add segment overrides for targeted experimentation

## Common Issues

- symptoms: "CreateExperiment returns ValidationException for metric goals"
  diagnosis: "Metric goal definition has invalid event pattern or missing required fields."
  resolution: "Verify metric goal includes: metricName, entityIdKey, valueKey, and eventPattern. The eventPattern must match the custom events sent via PutProjectEvents."

## Safety Ratings

```
safety_ratings:
  - "list-experiments: GREEN — read-only experiment enumeration"
  - "get-feature: GREEN — read-only feature details"
  - "CreateExperiment: YELLOW — creates experiment resource"
  - "start-experiment: YELLOW — begins experiment traffic allocation"
```

## Escalation Conditions

- Experiment creation fails with internal errors after fixing all parameters
- Metric goal definitions accepted but no metrics collected after experiment starts
- Experiment requires features or variations that do not exist
- Multiple experiments needed on the same feature simultaneously

## Data Sensitivity

- `aws evidently list-experiments`: LOW — returns experiment metadata
- `aws evidently get-feature`: LOW — returns feature configuration
- Experiment configuration: MEDIUM — reveals A/B testing strategy and metric goals

## Prohibited Actions

- NEVER start an experiment without verifying metric event collection is working
- NEVER create experiments with placeholder metric definitions in production
- NEVER delete a feature that has an active experiment
- NEVER create experiments without a clear hypothesis and success criteria

## Phase 3 — Rollback

1. If an experiment was created with wrong configuration, delete it before starting: `aws evidently delete-experiment --project <name> --experiment <experiment-name>`
2. Recreate with correct metric goals and treatments
3. If the experiment was already started, stop it: `aws evidently stop-experiment --project <name> --experiment <experiment-name> --desired-state CANCELLED`
4. Verify feature evaluations return to normal after experiment cleanup
5. Document the experiment creation issue and resolution

## Output Format

```yaml
root_cause: "experiment_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details and metric goal configuration>"
severity: HIGH
mitigation:
  immediate: "Fix experiment creation parameters"
  long_term: "Validate experiment configurations in staging before production"
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
