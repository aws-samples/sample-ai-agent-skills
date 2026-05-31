# Amazon Security Lake Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Security Lake problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Security Lake — lake creation and configuration, source ingestion, subscriber management, OCSF schema normalization, Athena query integration, cross-account and cross-region collection, rollup regions, custom sources, data retention lifecycle, and IAM/Lake Formation permissions.

### Activate When

- Lake creation or update failures
- Source ingestion not working or delayed
- Subscriber cannot access data
- OCSF schema normalization errors
- Athena queries returning no data or errors
- Cross-account log collection failures
- Rollup region sync issues
- Custom source delivery failures
- Data retention policy not applied
- Lake Formation permission errors
- General Security Lake errors without clear symptoms

---

## Skill Structure

```
securitylake-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-lake-creation.md
    ├── A2-lake-configuration.md
    ├── B1-source-configuration.md
    ├── B2-custom-sources.md
    ├── C1-subscriber-management.md
    ├── C2-subscriber-access.md
    ├── D1-ocsf-mapping.md
    ├── D2-schema-validation.md
    ├── E1-athena-integration.md
    ├── E2-query-failures.md
    ├── F1-cross-account-region.md
    ├── F2-rollup-regions.md
    ├── G1-data-retention.md
    ├── G2-lifecycle-management.md
    ├── H1-iam-roles.md
    ├── H2-lake-formation-permissions.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Lake Setup** | A1–A2 | Lake creation, configuration |
| **B — Sources** | B1–B2 | Source configuration, custom sources |
| **C — Subscribers** | C1–C2 | Subscriber management, access issues |
| **D — Schema** | D1–D2 | OCSF mapping, schema validation |
| **E — Query** | E1–E2 | Athena integration, query failures |
| **F — Cross-Account** | F1–F2 | Multi-account, rollup regions |
| **G — Retention** | G1–G2 | Data retention, lifecycle |
| **H — Permissions** | H1–H2 | IAM roles, Lake Formation |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `securitylake:*`, `lakeformation:*`, `glue:Get*`, `athena:*`, `s3:Get*`, `cloudtrail:LookupEvents`

---

## License

MIT-0
