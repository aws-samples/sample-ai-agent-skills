# Amazon DocumentDB Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon DocumentDB problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon DocumentDB when the console alone isn't enough — cluster health analysis, query performance debugging, connectivity troubleshooting, replication monitoring, MongoDB compatibility issues, change stream problems, backup/restore, and security configuration.

### Activate When

- Cluster creation failures
- Instance launch or health issues
- Storage capacity or growth problems
- Query performance degradation
- Missing or inefficient indexes
- Profiler analysis needed
- Connection failures or timeouts
- TLS certificate or configuration issues
- Connection pool exhaustion
- Read replica lag
- Global cluster replication issues
- MongoDB compatibility differences
- Driver connection or version issues
- Change stream configuration problems
- Change stream consumer lag or failures
- Snapshot creation failures
- Point-in-time restore issues
- Authentication or authorization errors
- Encryption configuration problems

---

## Skill Structure

```
documentdb-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cluster-creation-failures.md
    ├── A2-instance-issues.md
    ├── A3-storage-issues.md
    ├── B1-query-performance.md
    ├── B2-index-management.md
    ├── B3-profiler.md
    ├── C1-connection-failures.md
    ├── C2-tls-issues.md
    ├── C3-connection-pooling.md
    ├── D1-read-replicas.md
    ├── D2-global-clusters.md
    ├── E1-mongodb-differences.md
    ├── E2-driver-issues.md
    ├── F1-stream-configuration.md
    ├── F2-consumer-issues.md
    ├── G1-snapshot-failures.md
    ├── G2-pitr.md
    ├── H1-authentication.md
    ├── H2-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── documentdb-guardrails.md
    └── documentdb-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A3 | Creation failures, instance issues, storage |
| **B — Performance** | B1–B3 | Query performance, index management, profiler |
| **C — Connectivity** | C1–C3 | Connection failures, TLS, connection pooling |
| **D — Replication** | D1–D2 | Read replicas, global clusters |
| **E — Compatibility** | E1–E2 | MongoDB differences, driver issues |
| **F — Change Streams** | F1–F2 | Stream configuration, consumer issues |
| **G — Backup** | G1–G2 | Snapshot failures, PITR |
| **H — Security** | H1–H2 | Authentication, encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/documentdb-guardrails.md` covering DocumentDB-is-not-MongoDB distinction, TLS requirement, change stream retention limits, index build write blocking, elastic cluster differences, parameter group behavior, backup method restrictions, aggregation pipeline limitations, storage auto-scaling, instance class selection, connection limits, and engine version compatibility.

---

## Investigation Workflow

1. **Triage** — Describe clusters/instances, check CloudWatch metrics, identify symptoms
2. **Deep Dive** — Examine parameter groups, events, endpoints, server status, current operations
3. **Detailed** — Inspect CloudTrail events, snapshots, global clusters, replication lag, KMS keys

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `rds:Describe*` (DocumentDB uses RDS API namespace), `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `ec2:Describe*`, `kms:DescribeKey`
- mongo shell or mongosh for direct cluster diagnostics (recommended)
- TLS certificate bundle (rds-combined-ca-bundle.pem)

---

## Usage Examples

```
# Describe DocumentDB cluster
aws docdb describe-db-clusters --db-cluster-identifier my-docdb-cluster

# List cluster instances
aws docdb describe-db-instances --filters Name=db-cluster-id,Values=my-docdb-cluster

# Check cluster events
aws docdb describe-events --source-type db-cluster --duration 1440

# Connect with mongo shell (TLS required)
mongo --tls --host my-cluster.cluster-xxxx.us-east-1.docdb.amazonaws.com:27017 --tlsCAFile rds-combined-ca-bundle.pem --username myuser --password mypass
```

---

## License

MIT-0
