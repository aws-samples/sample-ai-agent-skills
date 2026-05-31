# AWS HealthImaging Diagnostics Skill

Agent skill for investigating and troubleshooting AWS HealthImaging problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS HealthImaging — datastore creation, DICOM import jobs, image set retrieval, metadata queries, copy/deletion operations, access control, encryption, and integration issues.

### Activate When

- Datastore creation failures
- Datastore configuration issues
- DICOM import job failures
- DICOM validation errors
- Image set retrieval problems
- Metadata query issues
- Copy job failures
- Deletion errors
- Access control problems
- Encryption configuration issues
- Integration issues
- General HealthImaging errors

---

## Skill Structure

```
healthimaging-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-datastore-creation.md
    ├── A2-datastore-configuration.md
    ├── B1-import-job-failures.md
    ├── B2-dicom-validation-errors.md
    ├── C1-image-set-retrieval.md
    ├── C2-metadata-queries.md
    ├── D1-copy-job-failures.md
    ├── D2-deletion-issues.md
    ├── E1-access-control.md
    ├── E2-encryption-configuration.md
    ├── F1-integration-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Datastore** | A1–A2 | Creation, configuration |
| **B — Import** | B1–B2 | Import job failures, DICOM validation |
| **C — Retrieval** | C1–C2 | Image set retrieval, metadata queries |
| **D — Operations** | D1–D2 | Copy job failures, deletion issues |
| **E — Security** | E1–E2 | Access control, encryption |
| **F — Integration** | F1 | Integration issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `medical-imaging:*`, `s3:GetObject`, `s3:PutObject`, `kms:Describe*`, `iam:Get*` permissions

---

## License

MIT-0
