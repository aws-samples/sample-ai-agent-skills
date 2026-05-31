# Amazon CloudWatch Evidently Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CloudWatch Evidently problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudWatch Evidently — project creation, feature flags, feature evaluations, launches, experiments, segments, metric definitions, and data export.

### Activate When

- Project creation failures
- Project configuration issues
- Feature creation errors
- Feature evaluation problems
- Launch configuration failures
- Launch scheduling issues
- Experiment creation failures
- Experiment results analysis problems
- Segment configuration issues
- Metric definition errors
- S3 data export failures
- General Evidently errors

---

## Skill Structure

```
evidently-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-project-creation-failures.md
    ├── A2-project-configuration.md
    ├── B1-feature-creation-errors.md
    ├── B2-feature-evaluation-issues.md
    ├── C1-launch-configuration-errors.md
    ├── C2-launch-scheduling-issues.md
    ├── D1-experiment-creation-failures.md
    ├── D2-experiment-results-analysis.md
    ├── E1-segment-configuration.md
    ├── E2-metric-definition-errors.md
    ├── F1-s3-data-export.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Project Setup** | A1–A2 | Creation failures, configuration |
| **B — Features** | B1–B2 | Creation errors, evaluation issues |
| **C — Launches** | C1–C2 | Configuration errors, scheduling issues |
| **D — Experiments** | D1–D2 | Creation failures, results analysis |
| **E — Segments & Metrics** | E1–E2 | Segment configuration, metric definition errors |
| **F — Data Export** | F1 | S3 data export |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `evidently:*`, `cloudwatch:Get*`, `s3:GetBucketPolicy`, `logs:Describe*`, `iam:Get*` permissions

---

## License

MIT-0
