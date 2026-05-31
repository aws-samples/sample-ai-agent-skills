# Amazon Comprehend Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Comprehend problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Comprehend — entity recognition, sentiment analysis, custom classifiers, custom entity recognition, async batch processing, PII detection, and topic modeling.

### Activate When

- Entity recognition errors or missing entities
- Sentiment analysis inaccuracies
- Custom classifier training failures
- Custom entity recognizer problems
- Async batch job failures
- PII detection or redaction issues
- Language detection errors
- General Comprehend errors

---

## Skill Structure

```
comprehend-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-entity-recognition.md
    ├── A2-custom-entities.md
    ├── B1-sentiment-analysis.md
    ├── B2-targeted-sentiment.md
    ├── C1-classifier-training.md
    ├── C2-classifier-deployment.md
    ├── C3-classifier-inference.md
    ├── D1-pii-detection.md
    ├── D2-pii-redaction.md
    ├── E1-batch-processing.md
    ├── E2-job-management.md
    ├── F1-language-detection.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Entity Recognition** | A1–A2 | Built-in entities, custom entities |
| **B — Sentiment** | B1–B2 | Sentiment analysis, targeted sentiment |
| **C — Custom Classifiers** | C1–C3 | Training, deployment, inference |
| **D — PII Detection** | D1–D2 | PII detection, PII redaction |
| **E — Async Jobs** | E1–E2 | Batch processing, job management |
| **F — Language** | F1 | Language detection |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `comprehend:*`, `s3:GetObject`, `s3:PutObject`, `iam:PassRole`

---

## License

MIT-0
