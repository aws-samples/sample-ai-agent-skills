# IAM Access Analyzer Diagnostics Skill

Agent skill for investigating and troubleshooting IAM Access Analyzer problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for IAM Access Analyzer — analyzer creation, findings, external access, unused access, policy validation, policy generation, and archive rules.

### Activate When

- Analyzer creation failures
- Finding management issues
- External access not detected
- Unused access analysis problems
- Policy validation errors
- Policy generation failures
- Archive rule configuration issues
- General Access Analyzer errors

---

## Skill Structure

```
accessanalyzer-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-analyzer-creation.md
    ├── A2-analyzer-configuration.md
    ├── B1-finding-management.md
    ├── B2-finding-resolution.md
    ├── C1-external-access-detection.md
    ├── C2-cross-account-access.md
    ├── D1-unused-access-analysis.md
    ├── D2-unused-permissions.md
    ├── E1-validation-errors.md
    ├── E2-validation-best-practices.md
    ├── F1-generation-setup.md
    ├── F2-generation-results.md
    ├── G1-archive-rule-creation.md
    ├── G2-archive-management.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Analyzer** | A1–A2 | Creation, configuration |
| **B — Findings** | B1–B2 | Management, resolution |
| **C — External Access** | C1–C2 | Detection, cross-account |
| **D — Unused Access** | D1–D2 | Analysis, permissions |
| **E — Policy Validation** | E1–E2 | Errors, best practices |
| **F — Policy Generation** | F1–F2 | Setup, results |
| **G — Archive Rules** | G1–G2 | Creation, management |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `access-analyzer:*`, `iam:Get*`, `s3:GetBucketPolicy`, `organizations:Describe*` permissions

---

## License

MIT-0
