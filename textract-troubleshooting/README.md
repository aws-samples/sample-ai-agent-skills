# Amazon Textract Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Textract problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Textract — document text detection, form/table extraction, async multi-page processing, custom queries, expense analysis, and identity document analysis.

### Activate When

- Document text detection errors or poor results
- Form key-value extraction failures
- Table extraction issues
- Async job failures or timeouts
- Custom query extraction problems
- Expense analysis errors
- Identity document analysis failures
- Low confidence scores
- General Textract errors

---

## Skill Structure

```
textract-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-text-detection.md
    ├── A2-document-analysis.md
    ├── B1-form-extraction.md
    ├── B2-table-extraction.md
    ├── C1-async-jobs.md
    ├── C2-multipage.md
    ├── D1-custom-queries.md
    ├── D2-query-accuracy.md
    ├── E1-expense-analysis.md
    ├── E2-identity-documents.md
    ├── F1-document-quality.md
    ├── F2-confidence-tuning.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Document Analysis** | A1–A2 | Text detection, document analysis |
| **B — Forms & Tables** | B1–B2 | Form extraction, table extraction |
| **C — Async Jobs** | C1–C2 | Job management, multi-page processing |
| **D — Queries** | D1–D2 | Custom queries, query accuracy |
| **E — Specialized** | E1–E2 | Expense analysis, identity documents |
| **F — Quality** | F1–F2 | Document quality, confidence tuning |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `textract:*`, `s3:GetObject`, `sns:Publish` (for async), `cloudwatch:GetMetricStatistics`
- S3 bucket with documents for analysis

---

## License

MIT-0
