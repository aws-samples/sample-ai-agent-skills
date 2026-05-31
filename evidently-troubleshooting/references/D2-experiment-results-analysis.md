---
title: "D2 — Experiment Results Analysis"
description: "Diagnose experiment results analysis and interpretation issues"
status: active
severity: HIGH
triggers: ["no experiment results", "results not significant", "wrong experiment data", "GetExperimentResults error"]
owner: devops-agent
objective: "Resolve experiment results analysis issues including missing data and statistical significance"
context: "Experiment results use Bayesian inference. Results require sufficient sample size and metric events sent via PutProjectEvents."
---

## Phase 1 — Triage

MUST:
- Get experiment status: `aws evidently get-experiment --project <name> --experiment <experiment-name>`
- Get experiment results: `aws evidently get-experiment-results --project <name> --experiment <experiment-name> --metric-names <metric> --treatment-names <treatment> --report-names BayesianInference`
- Verify the experiment is in RUNNING or COMPLETED status

SHOULD:
- Check if metric events are being sent via PutProjectEvents
- Verify the metric event pattern matches the experiment's metric goal definition
- Check the sample size and confidence intervals in results

MAY:
- Check data delivery destination for raw experiment data
- Compare results across different time periods
- Verify entity ID consistency between evaluations and metric events

## Phase 2 — Remediate

MUST:
- If no results: verify metric events are being sent and match the metric goal pattern
- If results not significant: allow more time for data collection or increase traffic
- Ensure entity IDs in metric events match entity IDs in feature evaluations

SHOULD:
- Verify the application is calling PutProjectEvents with correct event structure
- Check that the experiment has been running long enough for the expected traffic volume
- Review confidence intervals before making decisions

MAY:
- Extend experiment duration if more data is needed
- Increase traffic allocation to the experiment

## Common Issues

- symptoms: "Experiment running but GetExperimentResults returns empty or no data"
  diagnosis: "Metric events may not be sent, or entity IDs don't match between evaluations and events."
  resolution: "Verify: 1) Application calls PutProjectEvents with metric data, 2) Event pattern matches metric goal, 3) Entity IDs are consistent between EvaluateFeature and PutProjectEvents calls."

## Safety Ratings

```
safety_ratings:
  - "get-experiment: GREEN — read-only experiment details"
  - "get-experiment-results: GREEN — read-only results retrieval"
  - "stop-experiment: RED — freezes results and stops data collection"
  - "update-experiment: YELLOW — modifies experiment configuration"
```

## Escalation Conditions

- Experiment results show unexpected patterns that contradict application metrics
- Results remain inconclusive after the planned experiment duration
- Metric events confirmed sent but not appearing in results
- Experiment results needed urgently for business decision

## Data Sensitivity

- `aws evidently get-experiment`: LOW — returns experiment configuration
- `aws evidently get-experiment-results`: MEDIUM — reveals A/B test results and statistical analysis
- Experiment data: MEDIUM — contains metric values and treatment performance

## Prohibited Actions

- NEVER declare a winner based on results with low confidence intervals
- NEVER stop an experiment to get final results before sufficient data is collected
- NEVER modify metric goals on a running experiment
- NEVER share experiment results externally without proper statistical context

## Phase 3 — Rollback

1. If the experiment was stopped prematurely, results are frozen and cannot be recovered — document the limitation
2. If metric event collection was fixed, allow additional time for data accumulation
3. If the experiment needs to restart, create a new experiment (stopped experiments cannot be restarted)
4. Document the results analysis issue and any decisions made
5. Verify feature evaluations return to normal after experiment completion

## Output Format

```yaml
root_cause: "experiment_results — <specific_cause>"
evidence:
  - type: results_analysis
    content: "<experiment results and statistical details>"
severity: HIGH
mitigation:
  immediate: "Fix metric event collection or extend experiment duration"
  long_term: "Implement metric event validation in CI/CD pipeline"
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
