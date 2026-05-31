# Aurora Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Aurora (MySQL and PostgreSQL) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Aurora when the console alone isn't enough — cluster creation failures, storage issues, performance degradation with Aurora-specific wait events, memory pressure, Aurora I/O model issues, connectivity and endpoint confusion, RDS Proxy issues, writer and reader failover, DNS propagation, replication lag, binlog and logical replication, Serverless v2 scaling and cold start, Global Database replication and failover, backup failures, backtrack (MySQL only), cloning, encryption, IAM database authentication, and migration from MySQL/PostgreSQL or via DMS.

### Activate When

- Aurora cluster creation failures
- Storage issues (auto-scaling, cannot shrink)
- Cluster endpoint or reader endpoint confusion
- Cluster or instance parameter group misconfiguration
- High CPU with Aurora-specific wait events
- Memory pressure on Aurora instances
- Aurora I/O model issues (Standard vs I/O-Optimized)
- Query performance degradation
- Connection failures or connection limit exhaustion
- Endpoint confusion (writer vs reader vs custom)
- RDS Proxy configuration or connectivity issues
- Connection pooling and limits
- Writer failover events or unexpected promotion
- Reader failover and routing issues
- DNS propagation delays after failover
- Reader replica lag
- Aurora MySQL binlog replication issues
- Aurora PostgreSQL logical replication issues
- Serverless v2 scaling problems or cold start
- Serverless v2 capacity allocation issues
- Global Database replication lag
- Global Database planned failover
- Global Database unplanned failover (detach and promote)
- Backup or snapshot failures
- Backtrack issues (Aurora MySQL only)
- Clone creation or behavior issues
- Encryption configuration (at-rest, in-transit)
- IAM database authentication issues
- DMS migration to Aurora
- MySQL or PostgreSQL migration to Aurora

---

## Skill Structure

```
aurora-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cluster-creation-failures.md
    ├── A2-storage-issues.md
    ├── A3-cluster-endpoint-issues.md
    ├── A4-cluster-parameter-groups.md
    ├── B1-high-cpu-waits.md
    ├── B2-memory-pressure.md
    ├── B3-io-issues.md
    ├── B4-query-performance.md
    ├── C1-connection-failures.md
    ├── C2-endpoint-confusion.md
    ├── C3-rds-proxy-issues.md
    ├── C4-connection-limits.md
    ├── D1-writer-failover.md
    ├── D2-reader-failover.md
    ├── D3-dns-propagation.md
    ├── E1-reader-lag.md
    ├── E2-mysql-binlog-replication.md
    ├── E3-postgresql-logical-replication.md
    ├── F1-scaling-issues.md
    ├── F2-cold-start.md
    ├── F3-capacity-allocation.md
    ├── G1-global-replication-lag.md
    ├── G2-planned-failover.md
    ├── G3-unplanned-failover.md
    ├── H1-backup-failures.md
    ├── H2-backtrack.md
    ├── H3-clone-issues.md
    ├── I1-encryption.md
    ├── I2-iam-database-auth.md
    ├── J1-dms-to-aurora.md
    ├── J2-mysql-postgresql-migration.md
    ├── Z1-general-troubleshooting.md
    ├── aurora-guardrails.md
    └── aurora-hallucination-patterns.yaml
```

---

## Runbook Library (34 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A4 | Creation failures, storage, endpoints, parameter groups |
| **B — Performance** | B1–B4 | CPU/waits, memory, I/O (Aurora model), query performance |
| **C — Connectivity** | C1–C4 | Connection failures, endpoint confusion, RDS Proxy, limits |
| **D — Failover** | D1–D3 | Writer failover, reader failover, DNS propagation |
| **E — Replication** | E1–E3 | Reader lag, MySQL binlog, PostgreSQL logical replication |
| **F — Serverless** | F1–F3 | Scaling issues, cold start, capacity allocation |
| **G — Global Database** | G1–G3 | Replication lag, planned failover, unplanned failover |
| **H — Backup & Recovery** | H1–H3 | Backup failures, backtrack (MySQL), clones |
| **I — Security** | I1–I2 | Encryption, IAM database authentication |
| **J — Migration** | J1–J2 | DMS to Aurora, MySQL/PostgreSQL migration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Usage Examples

**Performance investigation:**
> "Aurora cluster my-prod-cluster has high CPU on the writer instance. Help me investigate."

**Failover issues:**
> "Our Aurora cluster just failed over and the application is still connecting to the old writer."

**Serverless scaling:**
> "Aurora Serverless v2 is not scaling up fast enough during peak traffic."

**Global Database:**
> "We need to perform a planned failover of our Aurora Global Database to the secondary region."

**Replication issues:**
> "Aurora reader replica lag is increasing on our PostgreSQL cluster."

---

## License

MIT-0
