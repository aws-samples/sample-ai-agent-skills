# Amazon Timestream Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Timestream problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Timestream when the console alone isn't enough — table creation, retention policies, write ingestion, rejected records, query errors, scheduled queries, memory/magnetic store configuration, Grafana integration, IoT Core integration, IAM permissions, and VPC endpoints.

### Activate When

- Table creation failures
- Retention policy misconfiguration
- Write ingestion errors or rejected records
- Batch write failures
- Query syntax errors or slow queries
- Scheduled query failures
- Memory store retention issues
- Magnetic store write problems
- Grafana plugin connection issues
- IoT Core rule action failures
- IAM access denied errors
- VPC endpoint configuration problems
- General Timestream errors without clear symptoms

---

## Skill Structure

```
timestream-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-retention-configuration.md
    ├── B1-write-failures.md
    ├── B2-rejected-records.md
    ├── B3-batch-write-errors.md
    ├── C1-query-failures.md
    ├── C2-query-performance.md
    ├── C3-scheduled-queries.md
    ├── D1-memory-store.md
    ├── D2-magnetic-store.md
    ├── E1-grafana-integration.md
    ├── E2-iot-core-integration.md
    ├── F1-iam-permissions.md
    ├── F2-vpc-endpoints.md
    ├── Z1-general-troubleshooting.md
    ├── timestream-guardrails.md
    └── timestream-hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Table** | A1–A2 | Creation failures, retention configuration |
| **B — Write** | B1–B3 | Write failures, rejected records, batch write errors |
| **C — Query** | C1–C3 | Query failures, performance, scheduled queries |
| **D — Memory/Magnetic** | D1–D2 | Memory store, magnetic store |
| **E — Integration** | E1–E2 | Grafana, IoT Core |
| **F — Security** | F1–F2 | IAM permissions, VPC endpoints |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/timestream-guardrails.md` covering memory vs magnetic retention units, late-arriving data rejection, append-only immutability, multi-measure vs single-measure records, scheduled query requirements, dimension indexing, Timestream for InfluxDB vs LiveAnalytics distinction, endpoint discovery requirement, magnetic store writes opt-in, query result pagination, data model planning, and CloudWatch metrics interpretation.

---

## Investigation Workflow

1. **Triage** — Check database/table status, retention configuration, recent errors
2. **Deep Dive** — Examine CloudWatch metrics, run diagnostic queries, check endpoints
3. **Detailed** — Review CloudTrail events, KMS configuration, integration settings

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `timestream:DescribeDatabase`, `timestream:DescribeTable`, `timestream:ListDatabases`, `timestream:ListTables`, `timestream:DescribeEndpoints`, `timestream:Select`, `cloudwatch:GetMetricStatistics`
- Timestream endpoint discovery access
- CloudWatch and CloudTrail access for monitoring

---

## Usage Examples

```
# Check database status
aws timestream-write describe-database --database-name my-database

# Check table retention settings
aws timestream-write describe-table --database-name my-database --table-name my-table

# Query recent data
aws timestream-query query --query-string "SELECT * FROM \"my-database\".\"my-table\" WHERE time > ago(1h) LIMIT 10"

# Check for errors
aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name UserErrors --dimensions Name=DatabaseName,Value=my-database --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 3600 --statistics Sum
```

---

## License

MIT-0
