# Athena Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Athena problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Athena when the AWS Console alone isn't enough — query execution failures, performance degradation, data format issues, partitioning problems, SerDe errors, workgroup configuration, federated queries, cost optimization, Glue Catalog integration, Lake Formation permissions, IAM access, encryption, and Athena for Apache Spark.

### Activate When

- Query execution fails with errors
- Queries timeout after 30 minutes
- SQL syntax errors on valid-looking queries
- Slow query performance or full table scans
- Data format mismatches (CSV vs JSON vs Parquet)
- Partition pruning not working
- SerDe deserialization errors
- Workgroup configuration issues
- Query result location not writable
- Federated query failures or connector errors
- High data scan costs
- Query result reuse not working
- Glue Data Catalog table definition issues
- Lake Formation permission denials
- IAM permission errors for Athena operations
- Encryption configuration problems
- Athena Spark session failures
- Notebook execution errors
- The user says something is wrong with Athena without naming specific symptoms

---

## Skill Structure

```
athena-troubleshooting/
├── SKILL.md                                  # Main skill definition and investigation workflow
├── README.md                                 # This file
└── references/
    ├── athena-guardrails.md                  # 12 anti-misdiagnosis guardrails
    ├── athena-hallucination-patterns.yaml    # 8 common LLM hallucination patterns
    ├── A1-query-failures.md
    ├── A2-query-timeout.md
    ├── A3-syntax-errors.md
    ├── A4-query-performance.md
    ├── B1-data-format-issues.md
    ├── B2-partitioning.md
    ├── B3-serde-errors.md
    ├── C1-workgroup-configuration.md
    ├── C2-query-result-location.md
    ├── D1-federated-query.md
    ├── D2-data-source-connectors.md
    ├── E1-data-scanned-optimization.md
    ├── E2-query-result-reuse.md
    ├── F1-glue-catalog.md
    ├── F2-lake-formation.md
    ├── G1-iam-permissions.md
    ├── G2-encryption.md
    ├── H1-athena-spark.md
    ├── H2-notebook-issues.md
    └── Z1-general-troubleshooting.md
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Query** | A1–A4 | Query failures, timeout, syntax errors, performance |
| **B — Data** | B1–B3 | Data format issues, partitioning, SerDe errors |
| **C — Workgroup** | C1–C2 | Workgroup configuration, query result location |
| **D — Federation** | D1–D2 | Federated query, data source connectors |
| **E — Cost** | E1–E2 | Data scanned optimization, query result reuse |
| **F — Integration** | F1–F2 | Glue Catalog, Lake Formation |
| **G — Security** | G1–G2 | IAM permissions, encryption |
| **H — Spark** | H1–H2 | Athena Spark, notebook issues |
| **Z — Catch-All** | Z1 | General Athena troubleshooting |

---

## Guardrails & Anti-Hallucination

The skill includes two guardrail files that prevent common misdiagnosis:

### Athena Guardrails (`athena-guardrails.md`)
12 rules covering: per-TB pricing (use partitioning/columnar), 30-min default timeout, workgroup controls, Glue Catalog is metastore, CTAS/INSERT INTO for ETL, federated queries use Lambda, result location must be writable, engine v3 vs v2 differences, partition projection config, SerDe must match format, S3 path cleanliness, no UPDATE/DELETE on regular tables.

### Hallucination Patterns (`athena-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about Athena, including:
- Persistent compute resources assumed (Athena is serverless)
- UPDATE/DELETE on regular tables (only Iceberg supports DML)
- Free queries or flat pricing (charged per TB scanned)
- Server-side configuration files (no infrastructure access)
- Unlimited query concurrency (per-account DML/DDL limits)
- Real-time query results (batch processing, not streaming)
- Native JDBC without driver (requires Athena JDBC driver)
- Automatic schema evolution (schema changes need table updates)

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using AWS CLI (`get-query-execution`, `get-work-group`, Glue table definitions), S3 path verification, and CloudTrail events. Classify the failure domain.

### Phase 2 — Remediate
Deep dive into the specific domain using targeted commands, query execution statistics, Glue catalog inspection, or IAM policy review. Apply the fix.

### Output Format
Every runbook produces structured YAML output:
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI with `athena`, `glue`, `s3`, `lakeformation`, `cloudtrail`, and `iam` permissions
- Access to Glue Data Catalog for table/partition inspection
- S3 access for data and result location verification
- Optional: Athena JDBC/ODBC driver for connectivity testing

---

## Usage Examples

### Query Failures
```
My Athena query is failing with "HIVE_CANNOT_OPEN_SPLIT" error when
querying a Parquet table in the "analytics" database. The table was
recently updated by a Glue ETL job. Investigate the data format and
table definition.
```

### Cost Optimization
```
Our Athena costs have tripled this month. We're scanning 50TB per day
across multiple workgroups. Help identify which queries are scanning
the most data and suggest optimization strategies.
```

### Federated Query Issues
```
Federated queries to our RDS PostgreSQL instance via the Athena
connector are failing with timeout errors. The Lambda connector
function is in a VPC. Investigate connectivity and configuration.
```

### General Triage
```
Something is wrong with our Athena setup but I'm not sure what.
Queries that worked last week are now failing. Run a general
investigation and follow whichever runbook matches the symptoms.
```

---

## License

MIT-0
