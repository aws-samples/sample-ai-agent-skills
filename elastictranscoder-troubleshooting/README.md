# Amazon Elastic Transcoder Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Elastic Transcoder problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Elastic Transcoder — pipeline creation, job failures, preset configuration, input/output format errors, notification delivery, permission issues, and watermark/thumbnail problems.

### Activate When

- Pipeline creation failures
- Pipeline configuration issues
- Job failures or errors
- Jobs stuck in progressing state
- Preset configuration errors
- Custom preset issues
- Input format not recognized
- Output format problems
- SNS notification failures
- IAM permission errors
- Watermark or thumbnail issues
- General Elastic Transcoder errors

---

## Skill Structure

```
elastictranscoder-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-pipeline-creation-failures.md
    ├── A2-pipeline-configuration.md
    ├── B1-job-failures.md
    ├── B2-job-stuck-progressing.md
    ├── C1-preset-errors.md
    ├── C2-custom-preset-issues.md
    ├── D1-input-format-errors.md
    ├── D2-output-format-issues.md
    ├── E1-notification-failures.md
    ├── E2-permission-errors.md
    ├── F1-watermark-thumbnail-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Pipelines** | A1–A2 | Creation failures, configuration |
| **B — Jobs** | B1–B2 | Job failures, stuck progressing |
| **C — Presets** | C1–C2 | Preset errors, custom presets |
| **D — Formats** | D1–D2 | Input format errors, output format issues |
| **E — Notifications & Permissions** | E1–E2 | SNS notifications, IAM permissions |
| **F — Media Features** | F1 | Watermarks and thumbnails |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `elastictranscoder:*`, `s3:GetObject`, `s3:PutObject`, `sns:Publish`, `iam:GetRole` permissions

---

## License

MIT-0
