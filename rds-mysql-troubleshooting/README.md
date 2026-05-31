# RDS MySQL Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon RDS for MySQL problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for RDS MySQL — instance launch failures, slow queries, InnoDB buffer pool tuning, lock waits and deadlocks, connectivity issues, parameter group configuration, read replica lag, Multi-AZ failover, backup/PITR, storage IOPS and autoscaling, RDS Proxy, encryption, and version upgrades.

### Activate When

- RDS MySQL instance launch failures
- Slow query performance or high CPU
- InnoDB buffer pool pressure or low hit ratio
- Lock waits, deadlocks, or long-running transactions
- Connection failures or max_connections exhaustion
- Parameter group misconfiguration
- Read replica lag or replication errors
- Multi-AZ failover events
- Backup failures or PITR issues
- Storage IOPS throttling or autoscaling problems
- RDS Proxy connection pooling issues
- Encryption configuration (KMS)
- Major or minor version upgrades (5.7→8.0)

---

## Skill Structure

```
rds-mysql-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-launch-failures.md
    ├── B1-slow-queries.md
    ├── B2-innodb-buffer-pool.md
    ├── B3-lock-waits.md
    ├── C1-connectivity.md
    ├── D1-parameter-groups.md
    ├── E1-read-replicas.md
    ├── E2-multi-az-failover.md
    ├── F1-backup-pitr.md
    ├── G1-iops-performance.md
    ├── G2-storage-autoscaling.md
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
| **B — Performance** | B1–B3 | Slow queries, InnoDB buffer pool, lock waits |
| **C — Connectivity** | C1 | Connection failures |
| **D — Parameters** | D1 | Parameter group issues |
| **E — Replication** | E1–E2 | Read replicas, Multi-AZ failover |
| **F — Backup** | F1 | Backup and PITR |
| **G — Storage** | G1–G2 | IOPS performance, storage autoscaling |
| **H — Upgrades** | H1 | Version upgrades |
| **I — Proxy** | I1 | RDS Proxy |
| **J — Security** | J1 | Encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Usage Examples

**Slow query investigation:**
> "Our RDS MySQL instance has queries taking over 30 seconds. Help me investigate."

**Buffer pool tuning:**
> "InnoDB buffer pool hit ratio is below 90% on our db.r5.xlarge instance."

**Replication lag:**
> "Read replica is showing 300 seconds of lag. What's causing it?"

---

## License

MIT-0
