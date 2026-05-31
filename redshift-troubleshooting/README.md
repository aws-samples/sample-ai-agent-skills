# Redshift Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Redshift problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Redshift provisioned clusters and Serverless when the AWS Console alone isn't enough — query performance, WLM queue tuning, storage management, COPY/UNLOAD operations, Spectrum external tables, data sharing, cluster operations, snapshots, security, and Serverless RPU configuration.

### Activate When

- Queries running slowly or timing out
- WLM queue wait times are high
- Lock contention blocking queries
- Query compilation taking too long
- Disk space running low or full
- Data skew across nodes
- VACUUM not completing or tables unsorted
- COPY command failures or data load errors
- UNLOAD command failures
- Spectrum external table query errors
- Node failures or hardware issues
- Resize operations failing or slow
- Maintenance window disruptions
- WLM queue misconfiguration
- Concurrency scaling not activating
- IAM or VPC access issues
- Encryption configuration problems
- Data sharing producer/consumer failures
- Cross-account data sharing issues
- Serverless RPU configuration problems
- Serverless endpoint access failures
- Snapshot creation failures
- Restore from snapshot issues
- The user says something is wrong with Redshift without naming specific symptoms

---

## Skill Structure

```
redshift-troubleshooting/
├── SKILL.md                                    # Main skill definition and investigation workflow
├── README.md                                   # This file
└── references/
    ├── redshift-guardrails.md                  # 12 anti-misdiagnosis guardrails
    ├── redshift-hallucination-patterns.yaml    # 8 common LLM hallucination patterns
    ├── A1-slow-queries.md
    ├── A2-queue-wait-times.md
    ├── A3-lock-contention.md
    ├── A4-query-compilation.md
    ├── B1-disk-full.md
    ├── B2-data-skew.md
    ├── B3-vacuum-issues.md
    ├── C1-copy-failures.md
    ├── C2-unload-errors.md
    ├── C3-spectrum-issues.md
    ├── D1-node-failures.md
    ├── D2-resize-operations.md
    ├── D3-maintenance-issues.md
    ├── E1-wlm-queue-config.md
    ├── E2-concurrency-scaling.md
    ├── F1-iam-vpc-config.md
    ├── F2-encryption-issues.md
    ├── G1-producer-consumer-sharing.md
    ├── G2-cross-account-sharing.md
    ├── H1-serverless-rpu-config.md
    ├── H2-serverless-endpoint-access.md
    ├── I1-snapshot-failures.md
    ├── I2-restore-issues.md
    └── Z1-general-troubleshooting.md
```

---

## Runbook Library (30 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Query Performance** | A1–A4 | Slow queries, queue wait times, lock contention, query compilation |
| **B — Storage** | B1–B3 | Disk full, data skew, VACUUM issues |
| **C — Data Loading** | C1–C3 | COPY failures, UNLOAD errors, Spectrum external tables |
| **D — Cluster Operations** | D1–D3 | Node failures, resize operations, maintenance |
| **E — WLM** | E1–E2 | Queue configuration, concurrency scaling |
| **F — Security** | F1–F2 | IAM/VPC configuration, encryption |
| **G — Data Sharing** | G1–G2 | Producer/consumer issues, cross-account sharing |
| **H — Serverless** | H1–H2 | RPU configuration, endpoint access |
| **I — Snapshots** | I1–I2 | Backup failures, restore issues |
| **Z — Catch-All** | Z1 | General Redshift troubleshooting |

---

## Guardrails & Anti-Hallucination

### Redshift Guardrails (`redshift-guardrails.md`)
12 rules covering: leader vs compute node roles, WLM concurrency limits, COPY vs INSERT for loading, distribution key impact, sort key maintenance, VACUUM requirements, Spectrum S3 scanning, concurrency scaling read-only, Serverless RPU model, cross-database limitations, maintenance window impact, RA3 managed storage.

### Hallucination Patterns (`redshift-hallucination-patterns.yaml`)
8 patterns that LLMs commonly get wrong about Redshift, including:
- SSH access to nodes (fully managed, no OS access)
- INSERT for bulk loading (COPY is required)
- Automatic VACUUM for all operations (manual VACUUM still needed)
- Unlimited concurrency (WLM controls query slots)
- Spectrum loads data into Redshift (it queries S3 in-place)
- Serverless has nodes (RPU-based, no node management)
- Cross-database writes (read-only cross-database queries)
- Classic resize is instant (takes hours, use elastic resize)

---

## Investigation Workflow

### Phase 1 — Triage
Collect initial evidence using AWS CLI, CloudWatch metrics, and Redshift system tables (STL_, SVL_, SVV_, STV_). Classify the failure domain.

### Phase 2 — Remediate
Deep dive into the specific domain using targeted system table queries, CloudWatch metrics, and CloudTrail events. Apply the fix.

### Output Format
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI with `redshift`, `redshift-data`, `redshift-serverless`, `cloudwatch`, `cloudtrail` permissions
- psql client (optional, for direct SQL access)
- S3 permissions (for COPY/UNLOAD debugging)
- IAM permissions for cluster/workgroup management

---

## Usage Examples

### Slow Queries
```
Queries on my Redshift cluster "analytics-prod" are taking 10x longer
than usual. The cluster has 4 ra3.xlplus nodes. Help me identify the
bottleneck — could be WLM, disk, skew, or missing VACUUM.
```

### COPY Failures
```
COPY command from S3 to my Redshift table is failing with "S3ServiceException:
Access Denied". The IAM role is attached to the cluster. Debug the
permission chain.
```

### Serverless Performance
```
My Redshift Serverless workgroup is showing high query times despite
having 128 RPU base capacity. Queries that used to take 5 seconds now
take 60 seconds. Investigate RPU utilization and scaling.
```

---

## License

MIT-0
