# FinSpace Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon FinSpace problems using structured runbooks.

---

## What This Skill Covers

Structured troubleshooting for Amazon FinSpace — environment configuration, datasets, managed kdb Insights, permissions, superuser access, and changeset management.

### Activate When

- Environment creation or configuration failures
- Dataset management issues
- Managed kdb cluster or database problems
- Permission or access control errors
- Superuser access issues
- Changeset creation or processing failures

---

## Skill Structure

```
finspace-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-environment-creation.md, A2-environment-config.md
    ├── B1-dataset-management.md, B2-dataset-views.md
    ├── C1-kdb-clusters.md, C2-kdb-databases.md
    ├── D1-access-control.md, D2-iam-permissions.md
    ├── E1-superuser.md, F1-changeset-management.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md, hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `finspace:*`, `s3:*`, `iam:*`

---

## License

MIT-0
