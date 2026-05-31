# RDS SQL Server Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon RDS for SQL Server problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for RDS SQL Server when the console alone isn't enough — instance launch failures, performance degradation with SQL Server wait statistics, memory pressure, I/O bottlenecks, tempdb contention, connectivity and Windows Authentication/Kerberos issues, SSL/TLS, parameter group tuning, native backup/restore to S3, snapshot restore, point-in-time recovery, read replica issues, Multi-AZ failover, native backup restore migration, DMS migration, collation problems, TDE encryption, SQL Server audit, login/user issues, version upgrades, and patching.

### Activate When

- RDS SQL Server instance launch failures
- Instance class or storage issues
- High CPU with SQL Server wait statistics
- Memory pressure (buffer pool, plan cache)
- I/O bottlenecks and storage latency
- Tempdb contention (PFS, GAM, SGAM, metadata)
- Connection failures or connection limit exhaustion
- Windows Authentication / Kerberos configuration
- SSL/TLS connection issues
- Parameter group misconfiguration
- SQL Server configuration tuning needs
- Max memory or max worker threads issues
- Native backup to S3 failures
- Snapshot restore problems
- Point-in-time recovery issues
- Read replica lag or creation failures
- Multi-AZ failover events (Always On AG)
- Native backup restore migration issues
- DMS migration issues
- Collation problems
- TDE encryption configuration issues (Enterprise only)
- SQL Server audit configuration
- Login and user permission issues
- Major or minor version upgrades
- Patching and maintenance window issues

---

## Skill Structure

```
rds-sqlserver-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-launch-failures.md
    ├── A2-instance-class-issues.md
    ├── A3-storage-issues.md
    ├── B1-high-cpu-waits.md
    ├── B2-memory-pressure.md
    ├── B3-io-bottleneck.md
    ├── B4-tempdb-contention.md
    ├── C1-connection-failures.md
    ├── C2-windows-auth-kerberos.md
    ├── C3-ssl-tls.md
    ├── D1-parameter-group-issues.md
    ├── D2-sqlserver-config.md
    ├── D3-max-memory-workers.md
    ├── E1-native-backup-s3.md
    ├── E2-snapshot-restore.md
    ├── E3-point-in-time-recovery.md
    ├── F1-read-replica-issues.md
    ├── F2-multi-az-failover.md
    ├── G1-native-backup-restore.md
    ├── G2-dms-issues.md
    ├── G3-collation-problems.md
    ├── H1-tde-issues.md
    ├── H2-sqlserver-audit.md
    ├── H3-login-user-issues.md
    ├── I1-version-upgrades.md
    ├── I2-patching.md
    ├── Z1-general-troubleshooting.md
    ├── rds-sqlserver-guardrails.md
    └── rds-sqlserver-hallucination-patterns.yaml
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance** | A1–A3 | Launch failures, instance class, storage |
| **B — Performance** | B1–B4 | CPU/wait stats, memory pressure, I/O, tempdb contention |
| **C — Connectivity** | C1–C3 | Connection failures, Windows Auth/Kerberos, SSL/TLS |
| **D — Parameters** | D1–D3 | Parameter groups, SQL Server config, max memory/workers |
| **E — Backup & Recovery** | E1–E3 | Native backup to S3, snapshot restore, PITR |
| **F — Replication** | F1–F2 | Read replicas, Multi-AZ failover |
| **G — Migration** | G1–G3 | Native backup restore, DMS, collation |
| **H — Security** | H1–H3 | TDE, SQL Server audit, login/user issues |
| **I — Maintenance** | I1–I2 | Version upgrades, patching |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Usage Examples

**Performance investigation:**
> "RDS SQL Server instance db-prod-sqlserver has high CPU. Help me investigate."

**Connection issues:**
> "Users are getting connection timeout errors connecting to our RDS SQL Server instance."

**Migration help:**
> "Native backup restore from S3 is failing on our RDS SQL Server instance."

**Encryption setup:**
> "We need to enable TDE on our RDS SQL Server Enterprise Edition instance."

---

## License

MIT-0
