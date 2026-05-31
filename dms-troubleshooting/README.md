# AWS DMS Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Database Migration Service problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS DMS when the console alone isn't enough — task failures, CDC replication lag, endpoint connectivity, replication instance sizing, schema conversion, table mapping, data validation, LOB handling, performance tuning, and security configuration.

### Activate When

- Migration task failures or errors
- Task stuck or not progressing
- CDC replication lag increasing
- Source endpoint connectivity issues
- Source-specific errors (Oracle, MySQL, PostgreSQL, SQL Server)
- Supplemental logging not configured
- Target endpoint errors
- Data type mapping failures
- Target apply errors or conflicts
- Replication instance undersized or overloaded
- Storage full on replication instance
- Schema conversion errors
- Table mapping issues
- Throughput bottlenecks
- Latency problems
- LOB handling failures or truncation
- Data validation mismatches
- Row count discrepancies
- IAM role misconfiguration
- VPC/subnet connectivity problems
- General DMS errors without clear symptoms

---

## Skill Structure

```
dms-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-task-failures.md
    ├── A2-task-stuck.md
    ├── A3-cdc-replication-lag.md
    ├── B1-source-endpoint-connectivity.md
    ├── B2-source-specific-errors.md
    ├── B3-supplemental-logging.md
    ├── C1-target-endpoint-errors.md
    ├── C2-data-type-mapping.md
    ├── C3-target-apply-errors.md
    ├── D1-replication-instance-sizing.md
    ├── D2-storage-full.md
    ├── E1-schema-conversion.md
    ├── E2-table-mapping.md
    ├── F1-throughput.md
    ├── F2-latency.md
    ├── F3-lob-handling.md
    ├── G1-data-validation-failures.md
    ├── G2-row-count-mismatch.md
    ├── H1-iam-roles.md
    ├── H2-vpc-subnet-config.md
    ├── Z1-general-troubleshooting.md
    ├── dms-guardrails.md
    └── dms-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Task** | A1–A3 | Task failures, task stuck, CDC replication lag |
| **B — Source** | B1–B3 | Source endpoint connectivity, source-specific errors, supplemental logging |
| **C — Target** | C1–C3 | Target endpoint errors, data type mapping, target apply errors |
| **D — Instance** | D1–D2 | Replication instance sizing, storage full |
| **E — Schema** | E1–E2 | Schema conversion, table mapping |
| **F — Performance** | F1–F3 | Throughput, latency, LOB handling |
| **G — Validation** | G1–G2 | Data validation failures, row count mismatch |
| **H — Security** | H1–H2 | IAM roles, VPC/subnet config |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/dms-guardrails.md` covering CDC supplemental logging requirements, LOB handling mode selection, replication instance sizing, storage monitoring, table mapping syntax, data validation behavior during CDC, Multi-AZ vs performance, VPC connectivity requirements, task settings tuning, schema conversion vs data migration distinction, source-specific prerequisites, and endpoint SSL/TLS configuration.

---

## Investigation Workflow

1. **Triage** — Check task status, endpoint connectivity, replication instance health, table statistics
2. **Deep Dive** — Examine CDC metrics, task assessments, connection tests, CloudWatch metrics
3. **Detailed** — Review CloudTrail events, task settings, individual assessment results

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `dms:Describe*`, `dms:TestConnection`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `ec2:DescribeSecurityGroups`, `ec2:DescribeSubnets`
- Database-level access for source/target verification (optional but recommended)
- CloudWatch Logs enabled on DMS tasks for detailed logging

---

## Usage Examples

```
# Check task status
aws dms describe-replication-tasks --filters Name=replication-task-id,Values=my-task

# Test endpoint connectivity
aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <endpoint-arn>

# Check table statistics
aws dms describe-table-statistics --replication-task-arn <task-arn>

# Check CDC latency
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencySource --dimensions Name=ReplicationInstanceIdentifier,Value=my-instance --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Average

# Check replication instance storage
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FreeStorageSpace --dimensions Name=ReplicationInstanceIdentifier,Value=my-instance --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Average
```

---

## License

MIT-0
