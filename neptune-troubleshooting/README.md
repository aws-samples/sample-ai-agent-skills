# Amazon Neptune Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Neptune graph database problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Neptune when the console alone isn't enough — cluster health analysis, query performance debugging (Gremlin and SPARQL), connectivity troubleshooting, replication monitoring, backup/restore, serverless scaling, data migration, and security configuration.

### Activate When

- Cluster creation failures
- Instance health issues or failures
- Storage capacity problems
- Gremlin query performance issues
- SPARQL query performance issues
- Bulk loading failures or slowness
- Connection failures or timeouts
- IAM authentication (SigV4) errors
- VPC endpoint connectivity issues
- Read replica lag or failures
- Global database replication issues
- Snapshot creation or restore failures
- Point-in-time recovery issues
- Neptune Serverless scaling problems
- Serverless capacity configuration
- Data loading or migration errors
- Format conversion issues

---

## Skill Structure

```
neptune-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cluster-creation-failures.md
    ├── A2-instance-issues.md
    ├── A3-storage.md
    ├── B1-query-performance.md
    ├── B2-gremlin-sparql-optimization.md
    ├── B3-bulk-loading.md
    ├── C1-connection-failures.md
    ├── C2-iam-auth.md
    ├── C3-vpc-endpoints.md
    ├── D1-read-replicas.md
    ├── D2-global-database.md
    ├── E1-snapshot-failures.md
    ├── E2-pitr.md
    ├── F1-serverless-scaling.md
    ├── F2-serverless-capacity.md
    ├── G1-data-loading.md
    ├── G2-format-conversion.md
    ├── Z1-general-troubleshooting.md
    ├── neptune-guardrails.md
    └── neptune-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A3 | Creation failures, instance issues, storage |
| **B — Performance** | B1–B3 | Query performance, Gremlin/SPARQL optimization, bulk loading |
| **C — Connectivity** | C1–C3 | Connection failures, IAM auth, VPC/endpoints |
| **D — Replication** | D1–D2 | Read replicas, global database |
| **E — Backup** | E1–E2 | Snapshot failures, PITR |
| **F — Serverless** | F1–F2 | Neptune Serverless scaling, capacity |
| **G — Migration** | G1–G2 | Data loading, format conversion |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/neptune-guardrails.md` covering VPC-only access, Gremlin vs SPARQL data model separation, bulk loader S3 requirements, IAM SigV4 authentication, Neptune Serverless NCU scaling, global database vs cross-region replicas, storage auto-scaling, instance class selection, query timeout limits, notebook integration, engine version compatibility, and parameter group behavior.

---

## Investigation Workflow

1. **Triage** — Describe clusters and instances, check CloudWatch metrics, verify cluster status endpoint
2. **Deep Dive** — Examine events, parameter groups, endpoints, run diagnostic queries
3. **Detailed** — Inspect CloudTrail events, snapshots, subnet groups, security groups, global clusters

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `neptune-db:*`, `rds:DescribeDB*`, `ec2:Describe*`, `s3:GetObject`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Network access to Neptune cluster endpoint (VPC, bastion, or VPN)
- curl or HTTP client for REST API endpoints (Gremlin/SPARQL/status)

---

## Usage Examples

```
# Describe Neptune cluster
aws neptune describe-db-clusters --db-cluster-identifier my-neptune-cluster

# Check cluster events
aws neptune describe-events --source-type db-cluster --duration 1440

# Check cluster status via HTTP
curl https://my-neptune-cluster.cluster-xxxx.us-east-1.neptune.amazonaws.com:8182/status

# Run Gremlin query
curl -X POST https://my-neptune-cluster.cluster-xxxx.us-east-1.neptune.amazonaws.com:8182/gremlin -d '{"gremlin":"g.V().limit(10)"}'

# Check bulk loader status
curl https://my-neptune-cluster.cluster-xxxx.us-east-1.neptune.amazonaws.com:8182/loader
```

---

## License

MIT-0
