# AWS Lake Formation Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Lake Formation problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Lake Formation when the console alone isn't enough — permission management, data location registration, catalog permissions, cross-account sharing, governed tables, blueprints, analytics service integration, data filters, and tag-based access control.

### Activate When

- Permission grant or revoke failures
- IAM vs Lake Formation permission conflicts
- Data location registration errors
- S3 path configuration issues
- Database or table permission problems
- Column-level security not working
- Cross-account sharing failures
- RAM integration issues with Lake Formation
- Governed table transaction errors
- Compaction issues
- Blueprint creation or execution failures
- Workflow execution problems
- Athena/Redshift/EMR access denied for LF-managed tables
- Glue integration issues
- Data filter configuration errors
- Tag-based access control not working
- General Lake Formation errors without clear symptoms

---

## Skill Structure

```
lakeformation-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-grant-failures.md
    ├── A2-revoke-issues.md
    ├── A3-iam-vs-lf-permissions.md
    ├── B1-data-location-registration.md
    ├── B2-s3-path-issues.md
    ├── C1-database-table-permissions.md
    ├── C2-column-level-security.md
    ├── D1-cross-account-sharing.md
    ├── D2-ram-integration.md
    ├── E1-transaction-issues.md
    ├── E2-compaction.md
    ├── F1-blueprint-failures.md
    ├── F2-workflow-issues.md
    ├── G1-athena-redshift-emr-access.md
    ├── G2-glue-integration.md
    ├── H1-data-filters.md
    ├── H2-tag-based-access-control.md
    ├── Z1-general-troubleshooting.md
    ├── lakeformation-guardrails.md
    └── lakeformation-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Permissions** | A1–A3 | Grant failures, revoke issues, IAM vs LF permissions |
| **B — Registration** | B1–B2 | Data location registration, S3 path issues |
| **C — Catalog** | C1–C2 | Database/table permissions, column-level security |
| **D — Cross-Account** | D1–D2 | Cross-account sharing, RAM integration |
| **E — Governed Tables** | E1–E2 | Transaction issues, compaction |
| **F — Blueprints** | F1–F2 | Blueprint failures, workflow issues |
| **G — Integration** | G1–G2 | Athena/Redshift/EMR access, Glue integration |
| **H — Security** | H1–H2 | Data filters, tag-based access control |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/lakeformation-guardrails.md` covering IAM-LF dual permission model, IAMAllowedPrincipals default behavior, data location registration requirement, cross-account RAM dependency, LF-Tag inheritance, column-level permission behavior, governed table transaction model, LF administrator privileges, blueprint execution role, data filter evaluation, hybrid permission mode, and catalog resource hierarchy.

---

## Investigation Workflow

1. **Triage** — Check LF settings, list permissions, verify data locations, check catalog
2. **Deep Dive** — Examine effective permissions, LF-Tags, cross-account shares
3. **Detailed** — Review CloudTrail events, data cell filters, RAM shares

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `lakeformation:*`, `glue:Get*`, `s3:GetBucketLocation`, `ram:Get*`, `iam:GetRole`, `cloudtrail:LookupEvents`
- Lake Formation administrator access for permission management
- Glue Data Catalog access

---

## Usage Examples

```
# Check Lake Formation settings and admins
aws lakeformation get-data-lake-settings

# List permissions for a principal
aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"arn:aws:iam::123456789012:role/MyRole"}'

# List registered data locations
aws lakeformation list-resources

# Check LF-Tags on a table
aws lakeformation get-resource-lf-tags --resource '{"Table":{"DatabaseName":"mydb","Name":"mytable"}}'

# Grant table permissions
aws lakeformation grant-permissions --principal '{"DataLakePrincipalIdentifier":"arn:aws:iam::123456789012:role/MyRole"}' --resource '{"Table":{"DatabaseName":"mydb","Name":"mytable"}}' --permissions SELECT
```

---

## License

MIT-0
