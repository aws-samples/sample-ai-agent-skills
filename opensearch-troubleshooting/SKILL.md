---
name: opensearch-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon OpenSearch Service
  problems (managed clusters and Serverless collections) by analyzing domain
  configurations, cluster health, indexing failures, search performance,
  storage pressure, JVM memory, shard management, snapshot/restore, access
  policies, VPC connectivity, Dashboards, UltraWarm/cold storage, and
  Serverless issues.
  Activate when: cluster health RED or YELLOW, search latency, indexing
  failures or throughput degradation, JVM memory pressure or GC pauses,
  disk watermark breaches, storage full, shard imbalance or unassigned
  shards, too many shards, mapping conflicts, bulk indexing errors, access
  policy denials, fine-grained access control (FGAC) issues, VPC
  connectivity problems, snapshot failures, restore issues, repository
  configuration, Dashboards access or visualization errors, UltraWarm or
  cold storage tier issues, ISM policy failures, index rollover problems,
  Serverless collection issues, Serverless capacity/scaling, split brain,
  dedicated master node issues, or the user says something is wrong with
  OpenSearch without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with OpenSearch Service, CloudWatch,
  CloudTrail, EC2 (for VPC/security groups), IAM, and optionally S3/KMS
  permissions. OpenSearch REST API access via curl recommended for direct
  cluster diagnostics.
---

# OpenSearch Diagnostics

## When to use

Any OpenSearch Service investigation where the console alone is insufficient — cluster health debugging, search performance analysis, indexing troubleshooting, shard management, storage pressure, JVM tuning, access policy resolution, VPC connectivity, snapshot/restore, Dashboards issues, UltraWarm/cold tier management, ISM lifecycle, or Serverless collection problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws opensearch describe-domain --domain-name <domain>
curl -XGET "https://<endpoint>/_cluster/health?pretty"
curl -XGET "https://<endpoint>/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent,node.role"
curl -XGET "https://<endpoint>/_cat/indices?v&s=health,index&h=health,status,index,pri,rep,docs.count,store.size"
aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name ClusterStatus.red --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum
aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name FreeStorageSpace --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum
aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name JVMMemoryPressure --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
curl -XGET "https://<endpoint>/_cat/shards?v&s=state,index&h=index,shard,prirep,state,docs,store,node,unassigned.reason"
curl -XGET "https://<endpoint>/_nodes/stats?pretty"
curl -XGET "https://<endpoint>/_cluster/settings?include_defaults=true&pretty"
curl -XGET "https://<endpoint>/_cat/allocation?v"
aws opensearch describe-domain-config --domain-name <domain>
aws cloudwatch get-metric-statistics --namespace AWS/ES --metric-name CPUUtilization --dimensions Name=DomainName,Value=<domain> Name=ClientId,Value=<account-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum
```

### Step 3 — Detailed investigation

```
aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.LogPublishingOptions'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=es.amazonaws.com --max-results 20
curl -XGET "https://<endpoint>/<index>/_settings?pretty"
curl -XGET "https://<endpoint>/<index>/_mapping?pretty"
curl -XGET "https://<endpoint>/_cluster/allocation/explain?pretty"
```

Read `references/opensearch-guardrails.md` before concluding on any OpenSearch issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-domain` | Full domain configuration and status |
| `describe-domain-config` | Domain configuration details including access policies |
| `_cluster/health` | Cluster health status, node count, shard counts |
| `_cat/nodes` | Node-level resource usage (heap, CPU, disk) |
| `_cat/indices` | Index health, doc counts, storage sizes |
| `_cat/shards` | Shard allocation, state, and unassigned reasons |
| `_cat/allocation` | Disk allocation per node |
| `_nodes/stats` | Detailed node statistics (JVM, OS, transport) |
| `_cluster/settings` | Cluster-level settings including watermarks |
| `_cluster/allocation/explain` | Why a shard is unassigned |
| `_nodes/hot_threads` | Identify CPU-intensive operations |
| `_tasks` | Running tasks (merges, recoveries, searches) |
| `_cat/recovery` | Shard recovery progress |
| `_plugins/_ism/explain/<index>` | ISM policy execution status |

## Gotchas: OpenSearch Service

- Cluster health RED means at least one primary shard is unassigned: RED does not mean the cluster is down. Searches and writes to indices with all primaries assigned still work. RED means at least one index has an unassigned primary shard, so that index is partially or fully unavailable. Check `_cat/shards?v&h=index,shard,prirep,state,unassigned.reason` to identify the affected index and reason.
- YELLOW means replicas are unassigned (single-node clusters are always YELLOW): YELLOW indicates all primary shards are assigned but at least one replica is not. Single-node domains are always YELLOW because replicas cannot be allocated to the same node as the primary. This is expected and not an error for single-node dev/test domains.
- JVM memory pressure > 80% causes GC pauses and potential OOM: OpenSearch uses Java heap for field data caches, query caches, and internal structures. When JVMMemoryPressure exceeds 80%, garbage collection becomes aggressive and causes latency spikes. Above 92%, the circuit breaker trips. Sustained pressure above 85% requires scaling or tuning.
- Shard count matters (aim for 10-50 GB per shard, max 1000 shards per node recommended): Oversized shards slow recovery and searches. Undersized shards waste resources with per-shard overhead (heap, file handles, cluster state). AWS recommends 10-50 GB per shard. Keep total shards per node under 1000. Total shard count = (primary shards) × (1 + replicas).
- Dedicated master nodes are critical for cluster stability (3 or 5, odd number): Dedicated master nodes manage cluster state, shard allocation, and index creation. Without them, data nodes handle master duties under load, risking instability. Always use 3 or 5 dedicated masters (odd number prevents split brain). Master nodes do not hold data.
- Storage watermarks (85% low, 90% high, 95% flood stage — blocks writes at flood): OpenSearch uses disk-based watermarks. At 85% (low), no new shards are allocated to the node. At 90% (high), shards are relocated away. At 95% (flood stage), all indices on the node become read-only (index.blocks.read_only_allow_delete). Writes are blocked until space is freed.
- Index State Management (ISM) policies for lifecycle: ISM automates index lifecycle operations (rollover, delete, snapshot, force merge, transition to warm/cold). Policies are attached to indices or index patterns. ISM runs on a configurable schedule (default 5 minutes). Failed ISM transitions require manual investigation via `_plugins/_ism/explain`.
- UltraWarm is read-only (cannot write to warm/cold): UltraWarm and cold storage tiers are for read-only data. You cannot index new documents into warm or cold indices. Data must be migrated from hot to warm using ISM or the migration API. Warm indices can be searched but not written to. Cold indices must be moved back to warm before searching.
- Access policies are resource-based (like S3 bucket policies): OpenSearch access policies are JSON resource-based policies attached to the domain. They control who can access the domain endpoint and which actions are allowed. They work alongside IAM policies. An explicit deny in either policy blocks access. IP-based conditions are common for non-VPC domains.
- VPC domains are not publicly accessible (no public endpoint): Domains deployed in a VPC have no public endpoint. Access requires being in the VPC, using VPN, VPC peering, Transit Gateway, or a proxy. Security groups control inbound access. VPC and public access cannot be changed after domain creation — you must create a new domain.
- OpenSearch Serverless has different API and scaling model: Serverless uses collections (not domains), OCUs (OpenSearch Compute Units) for capacity, and has different APIs. Serverless manages infrastructure automatically. Data access policies, network policies, and encryption policies replace domain-level settings. Not all OpenSearch features are available in Serverless.
- Fine-grained access control (FGAC) is separate from resource policy: FGAC provides user-level, role-level, and index-level permissions within the cluster. It uses an internal user database or SAML/Cognito. FGAC is independent of the domain access policy — both must allow access. FGAC is required for multi-tenant or least-privilege access patterns.
- Bulk indexing is much faster than individual document indexing: The `_bulk` API indexes thousands of documents in a single request, dramatically reducing overhead. Individual PUT requests have per-request overhead (HTTP connection, routing, refresh). Use bulk sizes of 5-15 MB. Too-large bulk requests can cause memory pressure and 429 errors.
- Cross-cluster search/replication have specific requirements: Cross-cluster search connects domains for federated queries. Cross-cluster replication copies indices between domains. Both require compatible versions, network connectivity, and proper access policies. Cross-cluster connections are unidirectional. VPC domains require VPC peering or Transit Gateway.
- Snapshot repository must be registered before backup/restore: Manual snapshots require registering an S3 repository first using the `_snapshot` API with an IAM role that has S3 access. Automated snapshots are taken daily by AWS but stored internally. Manual snapshots go to your S3 bucket. You cannot restore automated snapshots to a different domain — use manual snapshots for cross-domain restore.

## Anti-hallucination rules

1. Always cite specific domain configurations, cluster health output, `_cat` API results, CloudWatch metrics, or node stats as evidence.
2. Never confuse cluster health RED (unassigned primaries) with cluster down. RED clusters can still serve requests for indices with assigned primaries.
3. Never suggest writing to UltraWarm or cold storage indices. These tiers are strictly read-only. Data must be migrated from hot tier.
4. Never claim VPC domains have public endpoints. VPC domains are only accessible from within the VPC or via VPN/peering/proxy.
5. OpenSearch Serverless uses collections, OCUs, and different APIs — never apply managed domain concepts (shards, nodes, JVM) to Serverless.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 34 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster Health | A1-A4 | RED cluster, YELLOW cluster, split brain, master node issues |
| B — Performance | B1-B4 | Search latency, indexing throughput, JVM memory pressure, GC pauses |
| C — Storage | C1-C3 | Disk watermarks, storage full, UltraWarm/cold tier issues |
| D — Shards | D1-D3 | Unassigned shards, shard imbalance, too many shards |
| E — Indexing | E1-E3 | Indexing failures, mapping conflicts, bulk indexing errors |
| F — Access & Security | F1-F3 | Access policy issues, fine-grained access control, VPC connectivity |
| G — Snapshots | G1-G3 | Snapshot failures, restore issues, repository configuration |
| H — Dashboards | H1-H2 | Dashboards access, visualization errors |
| I — Serverless | I1-I2 | Collection issues, capacity/scaling |
| J — ISM & Lifecycle | J1-J2 | ISM policy failures, index rollover |
| Z — Catch-All | Z1 | General troubleshooting |
