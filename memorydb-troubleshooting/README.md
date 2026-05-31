# Amazon MemoryDB for Redis Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon MemoryDB for Redis problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon MemoryDB when the console alone isn't enough — cluster health analysis, performance debugging, connectivity troubleshooting, durability verification, scaling operations, backup/restore, security configuration, and MemoryDB vs ElastiCache differentiation.

### Activate When

- Cluster creation failures
- Node health issues or failures
- Shard configuration problems
- Latency spikes or high latency
- Throughput degradation
- Memory pressure or OOM errors
- Connection failures or timeouts
- ACL authentication errors
- Transaction log concerns
- Data persistence questions
- Vertical scaling (instance class change)
- Horizontal scaling (resharding)
- Snapshot creation failures
- Restore operation issues
- ACL management problems
- Encryption configuration issues

---

## Skill Structure

```
memorydb-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cluster-creation-failures.md
    ├── A2-node-issues.md
    ├── A3-shard-configuration.md
    ├── B1-latency.md
    ├── B2-throughput.md
    ├── B3-memory-pressure.md
    ├── C1-connection-failures.md
    ├── C2-acl-authentication.md
    ├── D1-transaction-log.md
    ├── D2-data-persistence.md
    ├── E1-vertical-scaling.md
    ├── E2-horizontal-scaling.md
    ├── F1-snapshot-failures.md
    ├── F2-restore-issues.md
    ├── G1-acl-management.md
    ├── G2-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── memorydb-guardrails.md
    └── memorydb-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A3 | Creation failures, node issues, shard configuration |
| **B — Performance** | B1–B3 | Latency, throughput, memory pressure |
| **C — Connectivity** | C1–C2 | Connection failures, ACL authentication |
| **D — Durability** | D1–D2 | Transaction log, data persistence vs ElastiCache |
| **E — Scaling** | E1–E2 | Vertical scaling, horizontal scaling (resharding) |
| **F — Backup** | F1–F2 | Snapshot failures, restore issues |
| **G — Security** | G1–G2 | ACL management, encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/memorydb-guardrails.md` covering MemoryDB durability vs ElastiCache, ACL-only authentication, always-on cluster mode, TLS requirement, transaction log write latency, snapshot cross-service compatibility, node type selection, shard and replica limits, connection limits, parameter group behavior, data eviction policies, and multi-AZ behavior.

---

## Investigation Workflow

1. **Triage** — Describe clusters, check CloudWatch metrics, identify cluster topology and health
2. **Deep Dive** — Examine events, parameter groups, ACLs, users, redis-cli diagnostics
3. **Detailed** — Inspect CloudTrail events, snapshots, subnet groups, security groups, replication lag

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `memorydb:*`, `ec2:Describe*`, `kms:DescribeKey`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- redis-cli with TLS support for direct node diagnostics (recommended)
- CloudWatch metrics enabled for MemoryDB

---

## Usage Examples

```
# Describe MemoryDB cluster
aws memorydb describe-clusters --cluster-name my-memorydb-cluster

# Check cluster events
aws memorydb describe-events --source-type cluster --duration 1440

# List ACLs
aws memorydb describe-acls

# Connect with redis-cli (TLS required)
redis-cli -h clustercfg.my-memorydb-cluster.xxxx.memorydb.us-east-1.amazonaws.com -p 6379 --tls --user myuser --pass mypass
```

---

## License

MIT-0
