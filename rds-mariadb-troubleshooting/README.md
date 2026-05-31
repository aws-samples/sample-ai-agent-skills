# RDS MariaDB Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon RDS for MariaDB problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for RDS MariaDB — instance launch failures, slow queries, InnoDB tuning, lock waits, connectivity, parameter groups, replication, backup/PITR, storage IOPS, MariaDB-specific features (thread pool, system-versioned tables, Aria engine), encryption, and version upgrades.

### Activate When

- RDS MariaDB instance launch failures
- Slow query performance or high CPU
- InnoDB buffer pool pressure
- Lock waits or deadlocks
- Connection failures or exhaustion
- Parameter group misconfiguration
- Read replica lag or replication errors
- Multi-AZ failover events
- Backup failures or PITR issues
- Storage IOPS throttling
- MariaDB-specific feature issues
- Encryption configuration
- Version upgrades

---

## Skill Structure

```
rds-mariadb-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-launch-failures.md
    ├── B1-performance.md
    ├── B2-lock-waits.md
    ├── C1-connectivity.md
    ├── D1-parameter-groups.md
    ├── E1-read-replicas.md
    ├── E2-multi-az-failover.md
    ├── F1-backup-pitr.md
    ├── G1-storage-iops.md
    ├── H1-version-upgrades.md
    ├── I1-mariadb-features.md
    ├── J1-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance** | A1 | Launch failures |
| **B — Performance** | B1–B2 | Slow queries, lock waits |
| **C — Connectivity** | C1 | Connection failures |
| **D — Parameters** | D1 | Parameter group issues |
| **E — Replication** | E1–E2 | Read replicas, Multi-AZ |
| **F — Backup** | F1 | Backup and PITR |
| **G — Storage** | G1 | Storage and IOPS |
| **H — Upgrades** | H1 | Version upgrades |
| **I — Features** | I1 | MariaDB-specific features |
| **J — Security** | J1 | Encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
