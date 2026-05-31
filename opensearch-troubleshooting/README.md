# OpenSearch Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon OpenSearch Service problems (managed domains and Serverless collections) using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for OpenSearch Service when the console alone isn't enough — cluster health debugging, search performance analysis, indexing troubleshooting, shard management, storage pressure, JVM tuning, access policy resolution, VPC connectivity, snapshot/restore, Dashboards issues, UltraWarm/cold tier management, ISM lifecycle, and Serverless collection problems.

### Activate When

- Cluster health RED or YELLOW (unexpected)
- Search latency or slow queries
- Indexing failures or rejected requests
- Indexing throughput degradation
- JVM memory pressure above 80%
- Garbage collection pauses affecting performance
- Disk watermark breaches (85%, 90%, 95%)
- Storage full or approaching capacity
- Unassigned shards (primary or replica)
- Shard imbalance across nodes
- Too many shards per node
- Mapping conflicts or type mismatches
- Bulk indexing errors (429, 413, parse failures)
- Access policy denials (403 errors)
- Fine-grained access control (FGAC) permission issues
- VPC connectivity problems
- Snapshot creation failures
- Snapshot restore failures
- Snapshot repository registration issues
- Dashboards login or access problems
- Dashboards visualization or query errors
- UltraWarm or cold storage migration issues
- ISM policy execution failures
- Index rollover not triggering
- Serverless collection creation or access issues
- Serverless capacity or scaling concerns
- Split brain or master node instability
- Dedicated master node failures

---

## Skill Structure

```
opensearch-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-red-cluster.md
    ├── A2-yellow-cluster.md
    ├── A3-split-brain.md
    ├── A4-master-node-issues.md
    ├── B1-search-latency.md
    ├── B2-indexing-throughput.md
    ├── B3-jvm-memory-pressure.md
    ├── B4-gc-pauses.md
    ├── C1-disk-watermarks.md
    ├── C2-storage-full.md
    ├── C3-ultrawarm-cold-tier.md
    ├── D1-unassigned-shards.md
    ├── D2-shard-imbalance.md
    ├── D3-too-many-shards.md
    ├── E1-indexing-failures.md
    ├── E2-mapping-conflicts.md
    ├── E3-bulk-indexing-errors.md
    ├── F1-access-policy-issues.md
    ├── F2-fine-grained-access-control.md
    ├── F3-vpc-connectivity.md
    ├── G1-snapshot-failures.md
    ├── G2-restore-issues.md
    ├── G3-repository-configuration.md
    ├── H1-dashboards-access.md
    ├── H2-visualization-errors.md
    ├── I1-serverless-collection-issues.md
    ├── I2-serverless-capacity.md
    ├── J1-ism-policy-failures.md
    ├── J2-index-rollover.md
    ├── Z1-general-troubleshooting.md
    ├── opensearch-guardrails.md
    └── opensearch-hallucination-patterns.yaml
```

---

## Runbook Library (34 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster Health** | A1–A4 | RED cluster, YELLOW cluster, split brain, master node issues |
| **B — Performance** | B1–B4 | Search latency, indexing throughput, JVM memory pressure, GC pauses |
| **C — Storage** | C1–C3 | Disk watermarks, storage full, UltraWarm/cold tier issues |
| **D — Shards** | D1–D3 | Unassigned shards, shard imbalance, too many shards |
| **E — Indexing** | E1–E3 | Indexing failures, mapping conflicts, bulk indexing errors |
| **F — Access & Security** | F1–F3 | Access policy issues, fine-grained access control, VPC connectivity |
| **G — Snapshots** | G1–G3 | Snapshot failures, restore issues, repository configuration |
| **H — Dashboards** | H1–H2 | Dashboards access, visualization errors |
| **I — Serverless** | I1–I2 | Collection issues, capacity/scaling |
| **J — ISM & Lifecycle** | J1–J2 | ISM policy failures, index rollover |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/opensearch-guardrails.md` covering cluster health color interpretation, JVM memory pressure thresholds, shard sizing recommendations, dedicated master node requirements, storage watermark behavior, UltraWarm read-only constraint, access policy evaluation, VPC domain accessibility, Serverless vs managed differences, FGAC independence from resource policy, bulk indexing best practices, and snapshot repository prerequisites.

---

## Investigation Workflow

1. **Triage** — Collect domain config, cluster health, node stats, index listing, key CloudWatch metrics (ClusterStatus, FreeStorageSpace, JVMMemoryPressure)
2. **Deep Dive** — Examine shard allocation, node stats, cluster settings, disk allocation, domain configuration
3. **Detailed** — Slow logs, CloudTrail events, index settings/mappings, allocation explain API

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `es:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `ec2:DescribeSecurityGroups`, `ec2:DescribeSubnets`, `iam:GetRole`, `s3:ListBucket`, `kms:DescribeKey`
- curl or HTTP client for OpenSearch REST API access
- Network access to OpenSearch domain endpoint (VPC access via bastion, VPN, or peering if VPC domain)
- Slow logs enabled (index.search.slowlog and index.indexing.slowlog settings)

---

## Usage Examples

```
# Get domain configuration
aws opensearch describe-domain --domain-name my-domain

# Check cluster health
curl -XGET "https://search-my-domain-abc123.us-east-1.es.amazonaws.com/_cluster/health?pretty"

# Check node resource usage
curl -XGET "https://search-my-domain-abc123.us-east-1.es.amazonaws.com/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent,node.role"

# Check index health
curl -XGET "https://search-my-domain-abc123.us-east-1.es.amazonaws.com/_cat/indices?v&s=health,index"

# Check JVM memory pressure
aws cloudwatch get-metric-statistics --namespace AWS/ES \
  --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=my-domain Name=ClientId,Value=123456789012 \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Maximum

# Check unassigned shards
curl -XGET "https://search-my-domain-abc123.us-east-1.es.amazonaws.com/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state"
```

---

## License

MIT-0
