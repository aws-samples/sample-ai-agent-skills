# ElastiCache Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon ElastiCache problems (Redis and Memcached) using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for ElastiCache when the console alone isn't enough — connection debugging, performance analysis, replication troubleshooting, scaling operations, memory management, security configuration, backup/restore, cluster mode operations, Memcached-specific issues, and Global Datastore problems.

### Activate When

- Connection failures or refused connections
- Timeout issues (connect or command timeouts)
- DNS or endpoint resolution problems
- High CPU or EngineCPU utilization
- Memory pressure, evictions, or OOM errors
- High latency on GET/SET operations
- Slow commands identified in slow log
- Replication lag between primary and replicas
- Failover not triggering or taking too long
- Read replica sync failures
- Vertical scaling (node type change) issues
- Horizontal scaling (resharding) problems
- ElastiCache Serverless scaling or limits
- Eviction storms causing cache misses
- Memory fragmentation ratio too high
- maxmemory or maxmemory-reserved misconfiguration
- Redis AUTH token rotation failures
- Encryption in-transit or at-rest issues
- Security group or subnet group misconfiguration
- Backup failures or excessive backup duration
- Snapshot restore failures
- Cluster mode slot migration errors
- Resharding failures or stuck operations
- Cross-slot command errors (CROSSSLOT)
- Memcached auto-discovery not working
- Memcached consistent hashing issues
- Global Datastore cross-region replication lag
- Global Datastore failover problems

---

## Skill Structure

```
elasticache-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-connection-failures.md
    ├── A2-timeout-issues.md
    ├── A3-dns-endpoint-resolution.md
    ├── B1-high-cpu.md
    ├── B2-memory-pressure-evictions.md
    ├── B3-high-latency.md
    ├── B4-slow-commands.md
    ├── C1-replication-lag.md
    ├── C2-failover-issues.md
    ├── C3-read-replica-problems.md
    ├── D1-vertical-scaling.md
    ├── D2-horizontal-scaling.md
    ├── D3-serverless-scaling.md
    ├── E1-eviction-storms.md
    ├── E2-memory-fragmentation.md
    ├── E3-maxmemory-configuration.md
    ├── F1-auth-token-issues.md
    ├── F2-encryption-issues.md
    ├── F3-security-group-subnet.md
    ├── G1-backup-failures.md
    ├── G2-restore-issues.md
    ├── H1-slot-migration.md
    ├── H2-resharding-failures.md
    ├── H3-cross-slot-errors.md
    ├── I1-memcached-auto-discovery.md
    ├── I2-memcached-consistent-hashing.md
    ├── J1-global-datastore-replication-lag.md
    ├── J2-global-datastore-failover.md
    ├── Z1-general-troubleshooting.md
    ├── elasticache-guardrails.md
    └── elasticache-hallucination-patterns.yaml
```

---

## Runbook Library (32 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Connectivity** | A1–A3 | Connection failures, timeout issues, DNS/endpoint resolution |
| **B — Performance** | B1–B4 | High CPU, memory pressure/evictions, high latency, slow commands |
| **C — Replication** | C1–C3 | Replication lag, failover issues, read replica problems |
| **D — Scaling** | D1–D3 | Vertical scaling, horizontal scaling (resharding), ElastiCache Serverless |
| **E — Memory** | E1–E3 | Eviction storms, memory fragmentation, maxmemory configuration |
| **F — Security** | F1–F3 | AUTH token issues, encryption, security group/subnet |
| **G — Backup** | G1–G2 | Backup failures, restore issues |
| **H — Cluster Mode** | H1–H3 | Slot migration, resharding failures, cross-slot errors |
| **I — Memcached** | I1–I2 | Auto-discovery, consistent hashing |
| **J — Global Datastore** | J1–J2 | Replication lag, failover |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/elasticache-guardrails.md` covering maxmemory-reserved sizing, eviction policy selection, TLS creation-time-only constraint, Memcached feature limitations, cluster mode enabled vs disabled differences, failover DNS propagation timing, connection limit awareness, parameter group apply types, backup memory doubling, Global Datastore manual failover, ElastiCache Serverless limits, and subnet/security group requirements.

---

## Investigation Workflow

1. **Triage** — Collect cluster config, replication group status, CloudWatch metrics (CPU, memory, evictions, connections)
2. **Deep Dive** — Examine events, parameter groups, redis-cli INFO/SLOWLOG, client connections
3. **Detailed** — CloudTrail events, replication lag metrics, subnet/security group config, snapshots

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `elasticache:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `ec2:DescribeSecurityGroups`, `ec2:DescribeSubnets`, `kms:DescribeKey`, `sns:ListTopics`
- Redis CLI (`redis-cli`) for direct node diagnostics (recommended)
- VPC access to ElastiCache nodes (bastion host, VPN, or VPC peering)
- ElastiCache slow log enabled (Redis parameter: slowlog-log-slower-than)

---

## Usage Examples

```
# Get all cache clusters with node info
aws elasticache describe-cache-clusters --show-cache-node-info

# Check replication group status
aws elasticache describe-replication-groups --replication-group-id my-redis-cluster

# Check CPU and memory metrics
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache \
  --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=my-redis-001 \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Average

# Check evictions
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache \
  --metric-name Evictions --dimensions Name=CacheClusterId,Value=my-redis-001 \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Sum

# Redis CLI diagnostics
redis-cli -h my-redis-cluster.abc123.ng.0001.use1.cache.amazonaws.com -p 6379 INFO
redis-cli -h my-redis-cluster.abc123.ng.0001.use1.cache.amazonaws.com -p 6379 SLOWLOG GET 25
```

---

## License

MIT-0
