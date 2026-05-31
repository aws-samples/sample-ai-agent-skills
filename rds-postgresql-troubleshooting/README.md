# RDS PostgreSQL Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon RDS for PostgreSQL problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for RDS PostgreSQL — instance launch failures, vacuum/autovacuum tuning, table bloat, connection pooling, pg_stat analysis, connectivity, parameter groups, physical and logical replication, backup/PITR, extensions, RDS Proxy, encryption, and version upgrades.

### Activate When

- RDS PostgreSQL instance launch failures
- Autovacuum not running or table bloat
- Transaction ID wraparound warnings
- Connection exhaustion or pooling issues
- Slow queries or high CPU from pg_stat analysis
- Parameter group misconfiguration
- Read replica lag or replication errors
- Logical replication setup or failures
- Backup failures or PITR issues
- Extension installation or compatibility problems
- RDS Proxy connection pooling issues
- Encryption configuration (KMS)
- Major version upgrades (e.g., 14→16)

---

## Skill Structure

```
rds-postgresql-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-launch-failures.md
    ├── B1-vacuum-bloat.md
    ├── B2-connection-pooling.md
    ├── B3-pg-stat-analysis.md
    ├── C1-connectivity.md
    ├── D1-parameter-groups.md
    ├── E1-read-replicas.md
    ├── E2-logical-replication.md
    ├── F1-backup-pitr.md
    ├── G1-extensions.md
    ├── H1-version-upgrades.md
    ├── I1-rds-proxy.md
    ├── J1-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance** | A1 | Launch failures |
| **B — Performance** | B1–B3 | Vacuum/bloat, connection pooling, pg_stat |
| **C — Connectivity** | C1 | Connection failures |
| **D — Parameters** | D1 | Parameter group issues |
| **E — Replication** | E1–E2 | Read replicas, logical replication |
| **F — Backup** | F1 | Backup and PITR |
| **G — Extensions** | G1 | Extension management |
| **H — Upgrades** | H1 | Version upgrades |
| **I — Proxy** | I1 | RDS Proxy |
| **J — Security** | J1 | Encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Usage Examples

**Vacuum investigation:**
> "Our RDS PostgreSQL tables have millions of dead tuples and autovacuum isn't keeping up."

**Connection pooling:**
> "We're hitting max_connections on our db.r6g.large PostgreSQL instance."

**Logical replication:**
> "Logical replication slot is growing and WAL is filling up disk."

---

## License

MIT-0
