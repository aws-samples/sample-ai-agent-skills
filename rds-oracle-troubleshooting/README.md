# RDS Oracle Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon RDS for Oracle problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for RDS Oracle when the console alone isn't enough — instance launch failures, performance degradation with Oracle wait events, SGA/PGA memory pressure, I/O bottlenecks, connectivity and TNS errors, parameter group tuning, backup and recovery, Multi-AZ failover, read replica issues, Data Pump and DMS migration, TDE encryption, SSL/native network encryption, Oracle audit, version upgrades, and patching.

### Activate When

- RDS Oracle instance launch failures
- Instance class or storage issues
- High CPU with Oracle wait events
- SGA/PGA memory pressure
- I/O bottlenecks and storage latency
- Oracle-specific wait events (enqueue, latch, log file sync)
- Connection failures or connection limit exhaustion
- TNS resolution errors
- Listener issues
- Parameter group misconfiguration
- Oracle init parameter tuning needs
- Session or process limit exceeded
- Automated backup failures
- Snapshot restore problems
- Point-in-time recovery issues
- Read replica lag or creation failures (Active Data Guard)
- Multi-AZ failover events
- Data Pump import/export failures
- DMS migration issues
- Character set or NLS problems
- TDE encryption configuration issues
- SSL or native network encryption problems
- Oracle audit configuration
- Major or minor version upgrades
- Patching and maintenance window issues

---

## Skill Structure

```
rds-oracle-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-launch-failures.md
    ├── A2-instance-class-issues.md
    ├── A3-storage-issues.md
    ├── B1-high-cpu-wait-events.md
    ├── B2-memory-pressure.md
    ├── B3-io-bottleneck.md
    ├── B4-oracle-specific-waits.md
    ├── C1-connection-failures.md
    ├── C2-tns-issues.md
    ├── C3-listener-issues.md
    ├── D1-parameter-group-issues.md
    ├── D2-oracle-init-parameters.md
    ├── D3-session-process-limits.md
    ├── E1-backup-failures.md
    ├── E2-snapshot-restore.md
    ├── E3-point-in-time-recovery.md
    ├── F1-read-replica-issues.md
    ├── F2-multi-az-failover.md
    ├── G1-data-pump-issues.md
    ├── G2-dms-issues.md
    ├── G3-character-set-problems.md
    ├── H1-tde-issues.md
    ├── H2-ssl-native-encryption.md
    ├── H3-oracle-audit.md
    ├── I1-version-upgrades.md
    ├── I2-patching.md
    ├── Z1-general-troubleshooting.md
    ├── rds-oracle-guardrails.md
    └── rds-oracle-hallucination-patterns.yaml
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Instance** | A1–A3 | Launch failures, instance class, storage |
| **B — Performance** | B1–B4 | CPU/wait events, SGA/PGA memory, I/O, Oracle waits |
| **C — Connectivity** | C1–C3 | Connection failures, TNS, listener |
| **D — Parameters** | D1–D3 | Parameter groups, init parameters, session/process limits |
| **E — Backup & Recovery** | E1–E3 | Backup failures, snapshot restore, PITR |
| **F — Replication** | F1–F2 | Read replicas (Active Data Guard), Multi-AZ failover |
| **G — Migration** | G1–G3 | Data Pump, DMS, character set |
| **H — Security** | H1–H3 | TDE, SSL/native encryption, Oracle audit |
| **I — Maintenance** | I1–I2 | Version upgrades, patching |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Usage Examples

**Performance investigation:**
> "RDS Oracle instance db-prod-oracle has high CPU. Help me investigate."

**Connection issues:**
> "Users are getting TNS:listener errors connecting to our RDS Oracle instance."

**Migration help:**
> "Data Pump import is failing on our RDS Oracle instance with ORA-39002."

**Encryption setup:**
> "We need to enable TDE on our RDS Oracle Enterprise Edition instance."

---

## License

MIT-0
