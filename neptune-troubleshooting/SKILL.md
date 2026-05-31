---
name: neptune-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Neptune
  problems by analyzing cluster configurations, instance health, query
  performance, CloudWatch metrics, and following structured runbooks.
  Activate when: cluster creation failures, instance issues, storage
  problems, Gremlin/SPARQL query performance, bulk loading failures,
  connection failures, IAM authentication errors, VPC endpoint issues,
  read replica lag, global database replication, snapshot failures,
  PITR issues, Neptune Serverless scaling, capacity problems, data
  loading errors, format conversion issues, or the user says something
  is wrong with Neptune without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Neptune, CloudWatch, CloudTrail,
  EC2, IAM, S3, and optionally Gremlin console or SPARQL endpoint for
  direct query diagnostics.
---

# Neptune Diagnostics

## When to use

Any Amazon Neptune investigation where the console alone is insufficient — cluster health analysis, query performance debugging, connectivity troubleshooting, replication monitoring, backup/restore, serverless scaling, data migration, or security configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws neptune describe-db-clusters
aws neptune describe-db-clusters --db-cluster-identifier <cluster-id>
aws neptune describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>
aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name CPUUtilization --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name FreeableMemory --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name GremlinRequestsPerSec --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws neptune describe-events --source-type db-cluster --duration 1440
aws neptune describe-events --source-type db-instance --duration 1440
aws neptune describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group>
aws neptune describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>
curl -X POST https://<cluster-endpoint>:8182/gremlin -d '{"gremlin":"g.V().count()"}'
curl -X POST https://<cluster-endpoint>:8182/sparql -d 'query=SELECT (COUNT(*) AS ?count) WHERE { ?s ?p ?o }'
curl https://<cluster-endpoint>:8182/status
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com --max-results 20
aws neptune describe-db-cluster-snapshots --db-cluster-identifier <cluster-id>
aws neptune describe-db-subnet-groups --db-subnet-group-name <subnet-group>
aws ec2 describe-security-groups --group-ids <sg-id>
aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name VolumeBytesUsed --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws neptune describe-global-clusters
```

Read `references/neptune-guardrails.md` before concluding on any Neptune issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-db-clusters` | Cluster configuration, endpoints, engine version |
| `describe-db-instances` | Instance status, class, and AZ placement |
| `describe-events` | Recent cluster and instance events |
| `describe-db-cluster-parameters` | Parameter group settings |
| `describe-db-cluster-endpoints` | Custom endpoint configuration |
| `/status` endpoint | Real-time cluster health and role |
| `/gremlin` endpoint | Gremlin query execution and profiling |
| `/sparql` endpoint | SPARQL query execution |
| `describe-global-clusters` | Global database topology |
| Bulk loader status | `curl https://<endpoint>:8182/loader/<load-id>` |

## Gotchas: Amazon Neptune

- Neptune is VPC-only — there is no public endpoint. All access must be from within the VPC, through a VPC endpoint, a bastion host, or an SSH tunnel. There is no option to enable public accessibility like RDS. This is a fundamental architectural constraint.
- Gremlin and SPARQL are different query languages on the same engine. Neptune supports both Apache TinkerPop Gremlin (property graph) and W3C SPARQL (RDF). Both run on the same cluster but use different data models. You cannot query Gremlin data with SPARQL or vice versa. Choose one model per use case.
- Bulk loader uses S3 for data ingestion. The Neptune bulk loader reads data from S3 buckets. It requires an IAM role with S3 access attached to the Neptune cluster. The loader supports CSV (Gremlin) and N-Triples/N-Quads/Turtle/RDF-XML (SPARQL). Direct data loading via queries is much slower.
- IAM authentication requires SigV4 signing. When IAM auth is enabled, all requests must be signed with AWS Signature Version 4. This applies to both Gremlin and SPARQL endpoints. Standard HTTP clients need SigV4 signing libraries. The Gremlin console requires special configuration for IAM auth.
- Neptune Serverless has a different scaling model. Neptune Serverless scales capacity in Neptune Capacity Units (NCUs). It scales based on workload demand between min and max NCU settings. Scaling is not instantaneous — there can be brief pauses during scale-up. Cost is based on NCUs consumed.
- No cross-region read replicas — use global database. Neptune does not support cross-region read replicas like Aurora. For cross-region disaster recovery or read scaling, use Neptune Global Database which replicates the entire cluster to secondary regions with typical lag under 1 second.
- Neptune notebooks provide integrated exploration. Neptune Workbench (Jupyter notebooks) provides an integrated environment for graph exploration with visualization. Notebooks connect directly to the cluster endpoint and support both Gremlin and SPARQL with built-in graph visualization.

## Anti-hallucination rules

1. Always cite specific cluster IDs, instance IDs, CloudWatch metrics, or query output as evidence.
2. Neptune is VPC-only. Never suggest enabling public access or connecting without VPC/tunnel.
3. Gremlin and SPARQL are separate data models. Never suggest cross-querying between them.
4. IAM auth requires SigV4. Never suggest simple username/password auth for Neptune.
5. No cross-region read replicas. Always recommend global database for cross-region needs.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1-A3 | Creation failures, instance issues, storage |
| B — Performance | B1-B3 | Query performance, Gremlin/SPARQL optimization, bulk loading |
| C — Connectivity | C1-C3 | Connection failures, IAM auth, VPC/endpoints |
| D — Replication | D1-D2 | Read replicas, global database |
| E — Backup | E1-E2 | Snapshot failures, PITR |
| F — Serverless | F1-F2 | Neptune Serverless scaling, capacity |
| G — Migration | G1-G2 | Data loading, format conversion |
| Z — Catch-All | Z1 | General troubleshooting |
