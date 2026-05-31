# Amazon WorkDocs Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon WorkDocs problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon WorkDocs — site creation, directory integration, upload/download failures, sync client issues, sharing permissions, collaboration, user management, admin controls, migration, storage limits, and API integration.

### Activate When

- Site creation failures
- Directory integration issues
- Upload or download failures
- Sync client problems
- Sharing permission errors
- Collaboration issues
- User management problems
- Admin control configuration
- Migration failures
- Storage limit issues
- API integration errors
- General WorkDocs errors

---

## Skill Structure

```
workdocs-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-site-creation-failures.md
    ├── A2-directory-integration.md
    ├── B1-upload-download-failures.md
    ├── B2-sync-client-issues.md
    ├── C1-sharing-permission-errors.md
    ├── C2-collaboration-issues.md
    ├── D1-user-management.md
    ├── D2-admin-controls.md
    ├── E1-migration-issues.md
    ├── E2-storage-limits.md
    ├── F1-api-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Sites** | A1–A2 | Creation failures, directory integration |
| **B — Content** | B1–B2 | Upload/download failures, sync client |
| **C — Sharing** | C1–C2 | Permission errors, collaboration |
| **D — Users** | D1–D2 | User management, admin controls |
| **E — Operations** | E1–E2 | Migration, storage limits |
| **F — Integration** | F1 | API integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `workdocs:*`, `ds:Describe*`, `iam:Get*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
