# Amazon Lookout for Equipment Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Lookout for Equipment problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Lookout for Equipment — dataset creation, data ingestion, model training, model evaluation, inference scheduling, inference execution, label management, retraining policies, S3 data access, IAM permissions, and data quality.

### Activate When

- Dataset creation failures
- Data ingestion errors
- Model training failures
- Model evaluation issues
- Inference scheduler errors
- Inference execution failures
- Label creation issues
- Retraining policy problems
- S3 data access errors
- IAM permission issues
- Data quality problems
- General Lookout for Equipment errors

---

## Skill Structure

```
lookout-equipment-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-dataset-creation-failures.md
    ├── A2-data-ingestion-errors.md
    ├── B1-model-training-failures.md
    ├── B2-model-evaluation-issues.md
    ├── C1-inference-scheduler-errors.md
    ├── C2-inference-execution-failures.md
    ├── D1-label-creation-issues.md
    ├── D2-retraining-policy.md
    ├── E1-s3-data-access.md
    ├── E2-iam-permissions.md
    ├── F1-data-quality-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Dataset** | A1–A2 | Creation failures, ingestion errors |
| **B — Model** | B1–B2 | Training failures, evaluation issues |
| **C — Inference** | C1–C2 | Scheduler errors, execution failures |
| **D — Labels & Retraining** | D1–D2 | Label creation, retraining policy |
| **E — Access** | E1–E2 | S3 data access, IAM permissions |
| **F — Data Quality** | F1 | Data quality issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `lookoutequipment:*`, `s3:Get*`, `s3:List*`, `iam:Get*`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricData` permissions

---

## License

MIT-0
