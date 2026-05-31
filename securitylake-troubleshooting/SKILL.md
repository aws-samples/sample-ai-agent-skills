---
name: securitylake-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Security Lake problems
  by analyzing lake creation, source configuration, subscriber management,
  OCSF schema mapping, Athena query integration, cross-account/region setup,
  rollup regions, custom sources, data retention, and IAM/Lake Formation
  permissions following structured runbooks. Activate when: lake creation
  failures, source ingestion issues, subscriber access problems, OCSF
  normalization errors, Athena query failures, cross-account collection
  issues, rollup region sync problems, custom source delivery failures,
  retention policy issues, Lake Formation permission errors, or the user
  says something is wrong with Security Lake without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with SecurityLake, S3, Glue, Athena,
  Lake Formation, IAM, CloudTrail, and Organizations permissions.
---

# Amazon Security Lake Diagnostics

## When to use

Any Amazon Security Lake investigation where the console alone is insufficient — lake creation and configuration, log source management, subscriber access, OCSF schema normalization, Athena query integration, cross-account and cross-region collection, rollup regions, custom sources, data retention lifecycle, or IAM and Lake Formation permission issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws securitylake list-data-lakes
aws securitylake get-data-lake-sources --accounts <account-id>
aws securitylake list-subscribers
aws securitylake list-log-sources
```

### Step 2 — Domain deep dive

```
aws securitylake get-subscriber --subscriber-id <id>
aws securitylake list-data-lake-exceptions --regions <region>
aws glue get-tables --database-name amazon_security_lake_glue_db_<region>
aws athena start-query-execution --query-string "SELECT * FROM amazon_security_lake_table_<region>_cloud_trail_mgmt_2_0 LIMIT 10" --result-configuration OutputLocation=s3://<bucket>/results/
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=securitylake.amazonaws.com --max-results 20
aws lakeformation list-permissions --principal DataLakePrincipal={DataLakePrincipalIdentifier=<subscriber-role-arn>}
aws s3 ls s3://<security-lake-bucket>/ --recursive --summarize
```

Read `references/guardrails.md` before concluding on any Security Lake issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-data-lakes` | Check lake status and configuration |
| `get-data-lake-sources` | Check source ingestion status |
| `list-subscribers` | Check subscriber configuration |
| `list-data-lake-exceptions` | Check for ingestion errors |
| Glue `get-tables` | Verify OCSF schema tables |
| Athena queries | Test data accessibility |
| Lake Formation | Check data permissions |
| CloudTrail | Audit configuration changes |

## Gotchas: Amazon Security Lake

- Security Lake uses OCSF (Open Cybersecurity Schema Framework) to normalize all log sources into a common schema. Native AWS sources (CloudTrail, VPC Flow Logs, Route 53, S3 Data Events, Lambda, EKS) are automatically normalized. Custom sources must conform to OCSF schema.
- Security Lake stores data in S3 using Apache Parquet format partitioned by region, account, and time. The S3 bucket is managed by Security Lake. Direct S3 access requires Lake Formation permissions, not just S3 bucket policies.
- Subscribers access data via S3 (query access) or SQS/EventBridge (data access notifications). Query subscribers get Lake Formation permissions to query via Athena. Data access subscribers get notified of new objects to pull.
- Cross-account collection requires AWS Organizations. The delegated administrator account manages Security Lake for the organization. Member accounts contribute logs automatically once sources are configured.
- Rollup regions aggregate data from contributing regions into a single region for centralized querying. Data is copied, not moved. Both regions retain data independently.
- Lake Formation permissions control access to Security Lake data, NOT IAM policies alone. Subscribers need Lake Formation grants on the Glue database and tables.

## Anti-hallucination rules

1. Always cite specific lake ARNs, subscriber IDs, or API responses as evidence.
2. Security Lake uses Lake Formation for access control, NOT just S3 policies.
3. OCSF schema is REQUIRED for custom sources. Never skip schema validation.
4. Cross-account requires Organizations delegated admin. Never assume direct setup.
5. Rollup regions COPY data, they do not move it. Never claim data is moved.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Lake Setup | A1–A2 | Lake creation, configuration |
| B — Sources | B1–B2 | Source configuration, custom sources |
| C — Subscribers | C1–C2 | Subscriber management, access issues |
| D — Schema | D1–D2 | OCSF mapping, schema validation |
| E — Query | E1–E2 | Athena integration, query failures |
| F — Cross-Account | F1–F2 | Multi-account, rollup regions |
| G — Retention | G1–G2 | Data retention, lifecycle |
| H — Permissions | H1–H2 | IAM roles, Lake Formation |
| Z — Catch-All | Z1 | General troubleshooting |
