---
name: lakeformation-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Lake Formation
  problems by analyzing permission grants, revoke issues, IAM vs LF
  permissions, data location registration, S3 path issues, database and
  table permissions, column-level security, cross-account sharing, RAM
  integration, governed tables, transactions, compaction, blueprint
  failures, workflow issues, Athena/Redshift/EMR access, Glue integration,
  data filters, tag-based access control, and following structured runbooks.
  Activate when: grant failures, revoke issues, IAM-LF permission conflicts,
  data location registration errors, S3 path problems, catalog permission
  issues, column-level security not working, cross-account sharing failures,
  governed table transaction errors, blueprint failures, query access denied
  from analytics services, data filter issues, or the user says something
  is wrong with Lake Formation without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Lake Formation, Glue, S3, IAM,
  RAM, CloudTrail, and optionally Athena, Redshift, and EMR permissions.
  Lake Formation administrator access required for permission management.
---

# AWS Lake Formation Diagnostics

## When to use

Any AWS Lake Formation investigation where the console alone is insufficient — permission grant/revoke failures, IAM vs Lake Formation permission conflicts, data location registration, catalog permissions, cross-account sharing, governed tables, blueprints, analytics service integration, or tag-based access control.

## Investigation workflow

### Step 1 — Collect and triage

```
aws lakeformation get-data-lake-settings
aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"<principal-arn>"}'
aws lakeformation list-resources
aws glue get-databases
```

### Step 2 — Domain deep dive

```
aws lakeformation get-effective-permissions-for-path --resource-arn <s3-arn>
aws lakeformation list-permissions --resource '{"Table":{"DatabaseName":"<db>","Name":"<table>"}}'
aws lakeformation get-resource-lf-tags --resource '{"Table":{"DatabaseName":"<db>","Name":"<table>"}}'
aws lakeformation search-tables-by-lf-tags --expression '[{"TagKey":"<key>","TagValues":["<value>"]}]'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lakeformation.amazonaws.com --max-results 20
aws lakeformation list-data-cells-filter
aws lakeformation get-data-lake-settings --query 'DataLakeSettings.DataLakeAdmins'
aws ram get-resource-shares --resource-owner SELF --resource-type glue:Table
```

Read `references/lakeformation-guardrails.md` before concluding on any Lake Formation issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `lakeformation get-data-lake-settings` | Check LF admins and default permissions |
| `lakeformation list-permissions` | List permissions for principals or resources |
| `lakeformation list-resources` | List registered data locations |
| `lakeformation get-effective-permissions-for-path` | Check effective permissions on S3 paths |
| `lakeformation list-data-cells-filter` | List data cell filters |
| `lakeformation get-resource-lf-tags` | Get LF-Tags on resources |
| `lakeformation search-tables-by-lf-tags` | Find tables by LF-Tag |
| `glue get-databases` / `glue get-tables` | Check Glue catalog resources |
| `ram get-resource-shares` | Check cross-account sharing via RAM |

## Gotchas: AWS Lake Formation

- Lake Formation permissions work ON TOP of IAM permissions. Both must allow access. LF permissions are the fine-grained layer; IAM provides the coarse-grained layer. A common mistake is granting LF permissions but forgetting IAM permissions, or vice versa.
- The IAMAllowedPrincipals group bypasses LF permissions. By default, new databases/tables grant ALL permissions to IAMAllowedPrincipals, which means IAM policies alone control access. To use LF permissions, you must remove IAMAllowedPrincipals grants.
- Data locations must be registered with Lake Formation. LF can only manage permissions on S3 locations that are registered. Unregistered locations fall back to IAM-only access control. Register with: aws lakeformation register-resource.
- Cross-account sharing uses RAM under the hood. LF cross-account grants create RAM resource shares. The recipient account must accept the RAM invitation (or use org sharing). Both accounts need LF and RAM permissions configured.
- LF-Tags provide attribute-based access control. Tags are key-value pairs assigned to databases, tables, and columns. Tag-based permissions grant access based on tag values, not individual resource names. This simplifies permission management at scale.
- Column-level permissions restrict which columns a principal can see. When column permissions are granted, the principal can only access the specified columns. This works with Athena, Redshift Spectrum, and EMR.
- Governed tables support ACID transactions. They use a different storage model and require specific permissions. Transaction conflicts can occur with concurrent writes. Compaction is needed to optimize storage.
- Lake Formation administrators have full permissions. LF admins can grant/revoke any permission and manage all resources. At least one admin must be configured. The IAM admin is NOT automatically an LF admin.

## Anti-hallucination rules

1. Always cite specific permission grants, resource ARNs, or API responses as evidence.
2. IAMAllowedPrincipals bypasses LF permissions. Never ignore this default behavior.
3. Data locations must be registered. Never assume LF manages unregistered S3 paths.
4. Cross-account sharing requires RAM. Never suggest direct cross-account LF grants without RAM.
5. LF permissions work WITH IAM, not instead of IAM. Never claim LF replaces IAM.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Permissions | A1-A3 | Grant failures, revoke issues, IAM vs LF permissions |
| B — Registration | B1-B2 | Data location registration, S3 path issues |
| C — Catalog | C1-C2 | Database/table permissions, column-level security |
| D — Cross-Account | D1-D2 | Cross-account sharing, RAM integration |
| E — Governed Tables | E1-E2 | Transaction issues, compaction |
| F — Blueprints | F1-F2 | Blueprint failures, workflow issues |
| G — Integration | G1-G2 | Athena/Redshift/EMR access, Glue integration |
| H — Security | H1-H2 | Data filters, tag-based access control |
| Z — Catch-All | Z1 | General troubleshooting |
