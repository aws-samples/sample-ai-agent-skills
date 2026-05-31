# Amazon Keyspaces (for Apache Cassandra) Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Keyspaces problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Keyspaces when the console alone isn't enough — table creation and schema management, CQL compatibility issues, read/write throttling, hot partition analysis, connectivity and TLS/SigV4 authentication, capacity mode selection, TTL behavior, PITR backup/restore, and Cassandra migration.

### Activate When

- Table creation failures or schema modification errors
- Capacity mode confusion (on-demand vs provisioned)
- Read or write throttling events
- Hot partition causing performance degradation
- Query performance issues or timeouts
- CQL compatibility errors or unsupported feature usage
- Connection failures to Keyspaces endpoint
- TLS certificate or SigV4 authentication errors
- On-demand vs provisioned capacity planning questions
- Auto-scaling not working or misconfigured
- TTL not expiring data as expected
- TTL configuration errors or value rejections
- PITR restore failures or slow restores
- Snapshot or backup questions
- Cassandra to Keyspaces migration issues
- CQLReplicator errors or replication lag
- General Keyspaces errors without clear symptoms

---

## Skill Structure

```
keyspaces-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-schema-issues.md
    ├── A3-capacity-mode.md
    ├── B1-read-write-throttling.md
    ├── B2-hot-partitions.md
    ├── B3-query-performance.md
    ├── C1-cql-compatibility.md
    ├── C2-unsupported-features.md
    ├── D1-connection-failures.md
    ├── D2-tls-sigv4-auth.md
    ├── E1-ondemand-vs-provisioned.md
    ├── E2-auto-scaling.md
    ├── F1-ttl-configuration.md
    ├── F2-expiration-behavior.md
    ├── G1-pitr.md
    ├── G2-snapshot-restore.md
    ├── H1-cassandra-migration.md
    ├── H2-cqlreplicator.md
    ├── Z1-general-troubleshooting.md
    ├── keyspaces-guardrails.md
    └── keyspaces-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Table** | A1–A3 | Creation failures, schema issues, capacity mode |
| **B — Performance** | B1–B3 | Read/write throttling, hot partitions, query performance |
| **C — CQL** | C1–C2 | CQL compatibility, unsupported features |
| **D — Connectivity** | D1–D2 | Connection failures, TLS/SigV4 auth |
| **E — Capacity** | E1–E2 | On-demand vs provisioned, auto-scaling |
| **F — TTL** | F1–F2 | TTL configuration, expiration behavior |
| **G — Backup** | G1–G2 | PITR, snapshot restore |
| **H — Migration** | H1–H2 | Cassandra migration, CQLReplicator |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/keyspaces-guardrails.md` covering Keyspaces-is-not-Cassandra distinction, TLS-mandatory connections, SigV4/service-specific credential authentication, partition key design criticality, CQL compatibility limitations, no UDFs/UDAs/materialized views, TTL maximum value enforcement, PITR continuous backup behavior, on-demand vs provisioned capacity differences, auto-scaling configuration requirements, ALLOW FILTERING restrictions, and no multi-region table support.

---

## Investigation Workflow

1. **Triage** — Check keyspace/table status, capacity mode, error metrics, throttling events
2. **Deep Dive** — Examine read/write latency, consumed capacity, auto-scaling targets, TTL/PITR config
3. **Detailed** — Review CloudTrail events, system errors, CQL query patterns, connection logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `cassandra:*`, `cloudwatch:GetMetricStatistics`, `iam:GetRole`, `cloudtrail:LookupEvents`, `application-autoscaling:Describe*`
- For CQL access: `cqlsh` with SigV4 authentication plugin or service-specific credentials
- TLS certificate (Starfield digital certificate) for client connections

---

## Usage Examples

```
# List all keyspaces
aws keyspaces list-keyspaces

# Check table configuration
aws keyspaces get-table --keyspace-name mykeyspace --table-name mytable

# Check throttling metrics
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=mytable --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Sum

# Check auto-scaling configuration
aws application-autoscaling describe-scalable-targets --service-namespace cassandra

# Connect via cqlsh with SigV4
cqlsh cassandra.<region>.amazonaws.com 9142 --ssl

# Check PITR status
aws keyspaces get-table --keyspace-name mykeyspace --table-name mytable --query 'PointInTimeRecovery'
```

---

## License

MIT-0
