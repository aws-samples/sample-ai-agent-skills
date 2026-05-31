# Clean Rooms Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Clean Rooms problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Clean Rooms when the console alone isn't enough — collaboration creation and membership, configured table setup, analysis rules, query execution and output, differential privacy, cryptographic computing, ML modeling, lookalike segments, IAM permissions, and data access control.

### Activate When

- Collaboration creation fails
- Membership invitation or acceptance issues
- Configured table creation errors
- Analysis rule configuration problems
- Query execution failures
- Query output not appearing in S3
- Differential privacy budget exhausted
- Cryptographic computing setup issues
- ML model training failures
- Lookalike segment generation problems
- IAM permission errors for Clean Rooms operations
- Data access or Glue catalog issues

---

## Skill Structure

```
cleanrooms-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-collaboration-creation.md
    ├── A2-membership.md
    ├── B1-configured-tables.md
    ├── B2-analysis-rules.md
    ├── C1-query-failures.md
    ├── C2-output-configuration.md
    ├── D1-differential-privacy.md
    ├── D2-cryptographic-computing.md
    ├── E1-ml-modeling.md
    ├── E2-lookalike-segments.md
    ├── F1-iam-permissions.md
    ├── F2-data-access.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (18 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Collaboration** | A1–A2 | Creation, membership |
| **B — Tables** | B1–B2 | Configured tables, analysis rules |
| **C — Queries** | C1–C2 | Query failures, output configuration |
| **D — Privacy** | D1–D2 | Differential privacy, cryptographic computing |
| **E — ML** | E1–E2 | ML modeling, lookalike segments |
| **F — Security** | F1–F2 | IAM permissions, data access |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
