# Amazon Translate Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Translate problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Translate — real-time translation, batch translation, custom terminology, parallel data customization, formality settings, and language detection.

### Activate When

- Translation errors or poor quality
- Custom terminology not applied
- Parallel data training failures
- Batch translation job failures
- Language pair not supported
- Formality or profanity masking issues
- General Translate errors

---

## Skill Structure

```
translate-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-translation-errors.md
    ├── A2-language-pairs.md
    ├── B1-terminology-creation.md
    ├── B2-terminology-application.md
    ├── C1-parallel-data.md
    ├── C2-custom-models.md
    ├── D1-batch-jobs.md
    ├── D2-format-issues.md
    ├── E1-formality.md
    ├── E2-profanity-masking.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Real-Time** | A1–A2 | Translation errors, language pairs |
| **B — Custom Terminology** | B1–B2 | Terminology creation, term application |
| **C — Parallel Data** | C1–C2 | Parallel data creation, custom models |
| **D — Batch Translation** | D1–D2 | Batch jobs, format issues |
| **E — Features** | E1–E2 | Formality, profanity masking |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `translate:*`, `s3:GetObject`, `s3:PutObject`, `iam:PassRole`

---

## License

MIT-0
