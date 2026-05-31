# Amazon Connect Contact Lens & Wisdom Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Connect Contact Lens and Wisdom problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Connect Contact Lens and Wisdom — Contact Lens enablement, real-time analytics, sentiment analysis, transcription, category rules, custom vocabulary, Wisdom knowledge bases, search, evaluation forms, screen recording, data lake export, S3 output, and third-party integrations.

### Activate When

- Contact Lens enablement issues
- Real-time analytics errors
- Sentiment analysis problems
- Transcript generation failures
- Category rule errors
- Custom vocabulary issues
- Wisdom knowledge base errors
- Wisdom search failures
- Evaluation form issues
- Screen recording errors
- Data lake export problems
- S3 output configuration issues
- Third-party integration failures
- General Contact Lens or Wisdom errors

---

## Skill Structure

```
connect-analytics-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-contact-lens-enablement.md
    ├── A2-real-time-analytics-errors.md
    ├── B1-sentiment-analysis-issues.md
    ├── B2-transcript-generation-failures.md
    ├── C1-category-rule-errors.md
    ├── C2-custom-vocabulary-issues.md
    ├── D1-wisdom-knowledge-base-errors.md
    ├── D2-wisdom-search-failures.md
    ├── E1-evaluation-form-issues.md
    ├── E2-screen-recording-errors.md
    ├── F1-data-lake-export.md
    ├── F2-s3-output-configuration.md
    ├── G1-third-party-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Enablement** | A1–A2 | Contact Lens enablement, real-time analytics |
| **B — Analysis** | B1–B2 | Sentiment analysis, transcription |
| **C — Rules** | C1–C2 | Category rules, custom vocabulary |
| **D — Wisdom** | D1–D2 | Knowledge bases, search |
| **E — Quality** | E1–E2 | Evaluation forms, screen recording |
| **F — Export** | F1–F2 | Data lake export, S3 output |
| **G — Integration** | G1 | Third-party integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `connect:*`, `wisdom:*`, `s3:Get*`, `kinesis:*`, `cloudwatch:*`, `iam:Get*` permissions
- Amazon Connect instance admin access for Contact Lens and Wisdom configuration

---

## License

MIT-0
