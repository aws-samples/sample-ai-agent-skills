# AWS Entity Resolution Advanced Diagnostics Skill

Agent skill for investigating and troubleshooting advanced AWS Entity Resolution problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for advanced Entity Resolution — matching workflow failures, ID mapping errors, schema mapping issues, provider service errors, rule-based matching tuning, ML matching accuracy, output configuration, data quality, cross-account workflows, privacy configuration, and Glue integration.

### Activate When

- Matching workflow failures
- ID mapping workflow errors
- Schema mapping issues
- Provider service errors
- Rule-based matching tuning problems
- ML matching accuracy concerns
- Output configuration errors
- Data quality issues
- Cross-account workflow failures
- Privacy configuration problems
- Glue integration errors
- General advanced Entity Resolution errors

---

## Skill Structure

```
entityresolution-advanced-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-matching-workflow-failures.md
    ├── A2-id-mapping-workflow-errors.md
    ├── B1-schema-mapping-issues.md
    ├── B2-provider-service-errors.md
    ├── C1-rule-based-matching-tuning.md
    ├── C2-ml-matching-accuracy.md
    ├── D1-output-configuration-errors.md
    ├── D2-data-quality-issues.md
    ├── E1-cross-account-workflows.md
    ├── E2-privacy-configuration.md
    ├── F1-glue-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Workflows** | A1–A2 | Matching failures, ID mapping errors |
| **B — Configuration** | B1–B2 | Schema mapping, provider services |
| **C — Matching** | C1–C2 | Rule-based tuning, ML accuracy |
| **D — Data** | D1–D2 | Output configuration, data quality |
| **E — Advanced** | E1–E2 | Cross-account, privacy |
| **F — Integration** | F1 | Glue integration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `entityresolution:*`, `glue:Get*`, `s3:GetObject`, `s3:PutObject`, `iam:Get*`, `cloudtrail:LookupEvents`, `kms:Describe*` permissions

---

## License

MIT-0
