---
name: evidently-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon CloudWatch Evidently
  problems by analyzing project creation, feature flags, feature evaluations,
  launches, experiments, segments, metrics, data export, and following
  structured runbooks.
  Activate when: project creation failures, feature flag issues, evaluation
  errors, launch configuration problems, experiment failures, segment issues,
  metric definition errors, data export problems, or the user says something
  is wrong with Evidently.
compatibility: >
  Requires AWS CLI or SDK access with evidently, cloudwatch, s3, logs,
  and iam permissions.
---

# Amazon CloudWatch Evidently Diagnostics

## When to use

Any CloudWatch Evidently investigation — project creation, feature flag management, feature evaluations, launch configuration, experiment setup, segment configuration, metric definitions, or data export.

## Investigation workflow

### Step 1 — Collect and triage

```
aws evidently list-projects --query 'projects[*].{Name:name,Status:status,Arn:arn}'
aws evidently get-project --project <name> --query '{Name:name,Status:status,DataDelivery:dataDelivery}'
aws evidently list-features --project <name> --query 'features[*].{Name:name,Status:status,EvaluationStrategy:evaluationStrategy}'
```

### Step 2 — Domain deep dive

```
aws evidently get-feature --project <name> --feature <feature-name>
aws evidently list-launches --project <name> --query 'launches[*].{Name:name,Status:status,Type:type}'
aws evidently list-experiments --project <name> --query 'experiments[*].{Name:name,Status:status,Type:type}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=evidently.amazonaws.com --max-results 20
aws evidently evaluate-feature --project <name> --feature <feature-name> --entity-id test-entity
aws evidently get-experiment-results --project <name> --experiment <experiment-name> --metric-names <metric> --treatment-names <treatment> --report-names BayesianInference
```

Read `references/guardrails.md` before concluding on any Evidently issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `evidently list-projects` | List all projects |
| `evidently get-project` | Get project details and data delivery config |
| `evidently list-features` | List features in a project |
| `evidently get-feature` | Get feature details and variations |
| `evidently evaluate-feature` | Test feature evaluation for an entity |
| `evidently list-launches` | List launches in a project |
| `evidently list-experiments` | List experiments in a project |

## Gotchas: CloudWatch Evidently

- Evidently projects are regional. Features, launches, and experiments exist within a project in a specific region.
- Feature evaluations use either client-side (local) or server-side evaluation. Client-side requires the SDK to download rules; server-side calls the EvaluateFeature API.
- Launches control gradual feature rollouts with traffic splits. They do NOT collect metrics — use experiments for metric analysis.
- Experiments require metric definitions and a baseline variation. Results use Bayesian inference and require sufficient sample size for statistical significance.
- Segments define subsets of users for targeted launches or experiments. They use pattern matching on entity attributes.
- Data delivery to S3 or CloudWatch Logs must be configured at the project level. It is not enabled by default.
- Feature variations can be boolean, string, long, or double types. The variation type cannot be changed after feature creation.

## Anti-hallucination rules

1. Always cite specific project names, feature names, or API responses as evidence.
2. Launches and experiments are different concepts. Launches control rollout; experiments measure impact. Never conflate them.
3. Feature evaluations depend on evaluation strategy (client-side vs server-side). Never assume one strategy.
4. Experiment results require statistical significance. Never claim results are conclusive without sufficient sample size.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Project Setup | A1–A2 | Creation failures, configuration |
| B — Features | B1–B2 | Creation errors, evaluation issues |
| C — Launches | C1–C2 | Configuration errors, scheduling issues |
| D — Experiments | D1–D2 | Creation failures, results analysis |
| E — Segments & Metrics | E1–E2 | Segment configuration, metric definition errors |
| F — Data Export | F1 | S3 data export |
| Z — Catch-All | Z1 | General troubleshooting |
