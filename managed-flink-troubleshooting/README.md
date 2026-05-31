# Amazon Managed Service for Apache Flink Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Managed Service for Apache Flink (formerly Kinesis Data Analytics) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Managed Service for Apache Flink — application creation, startup, Flink job failures, checkpointing, source/sink connectors, scaling, parallelism, state management, watermarks, VPC connectivity, IAM permissions, and Studio notebooks.

### Activate When

- Application creation failures
- Application start errors
- Flink job failures
- Checkpoint failures
- Source connector errors
- Sink connector errors
- Scaling issues
- Parallelism configuration problems
- State management issues
- Watermark problems
- VPC connectivity failures
- IAM permission errors
- Studio notebook issues
- General Managed Flink errors

---

## Skill Structure

```
managed-flink-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-application-creation-failures.md
    ├── A2-application-start-errors.md
    ├── B1-flink-job-failures.md
    ├── B2-checkpoint-failures.md
    ├── C1-source-connector-errors.md
    ├── C2-sink-connector-errors.md
    ├── D1-scaling-issues.md
    ├── D2-parallelism-configuration.md
    ├── E1-state-management.md
    ├── E2-watermark-issues.md
    ├── F1-vpc-connectivity.md
    ├── F2-iam-permissions.md
    ├── G1-studio-notebook-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Application** | A1–A2 | Creation failures, start errors |
| **B — Flink Jobs** | B1–B2 | Job failures, checkpoint failures |
| **C — Connectors** | C1–C2 | Source connectors, sink connectors |
| **D — Scaling** | D1–D2 | Scaling issues, parallelism configuration |
| **E — State & Time** | E1–E2 | State management, watermarks |
| **F — Infrastructure** | F1–F2 | VPC connectivity, IAM permissions |
| **G — Studio** | G1 | Studio notebook issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `kinesisanalyticsv2:*`, `kinesis:Describe*`, `s3:Get*`, `iam:Get*`, `cloudwatch:GetMetricStatistics`, `logs:FilterLogEvents` permissions

---

## License

MIT-0
