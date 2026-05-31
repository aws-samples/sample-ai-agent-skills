# Amazon Neptune Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Neptune issue.

## Guardrail 1: Neptune Is VPC-Only — No Public Endpoint
Neptune clusters can only be accessed from within a VPC. There is no option to enable public accessibility like RDS or Aurora. All access must go through the VPC — via a bastion host, VPN, AWS Direct Connect, VPC peering, or a VPC endpoint. Never suggest enabling public access on a Neptune cluster. If external access is needed, use an ALB/NLB proxy, API Gateway, or SSH tunnel.

## Guardrail 2: Gremlin and SPARQL Are Separate Data Models
Neptune supports both Gremlin (property graph) and SPARQL (RDF) on the same cluster, but they use completely different data models. Gremlin data (vertices, edges, properties) cannot be queried with SPARQL, and RDF triples cannot be queried with Gremlin. Choose one data model per use case. Loading data in one format does not make it available in the other.

## Guardrail 3: Bulk Loader Requires S3 and IAM Role
The Neptune bulk loader reads data exclusively from S3. It requires an IAM role attached to the Neptune cluster with s3:GetObject and s3:ListBucket permissions. The S3 bucket must be in the same region as the Neptune cluster. The loader supports CSV format for Gremlin and N-Triples/N-Quads/Turtle/RDF-XML for SPARQL. Direct loading via individual queries is orders of magnitude slower.

## Guardrail 4: IAM Authentication Uses SigV4 — Not Username/Password
When IAM authentication is enabled on Neptune, all HTTP requests must be signed with AWS Signature Version 4. There is no username/password authentication. The Gremlin console, SPARQL clients, and HTTP clients all need SigV4 signing. Use the AWS SDK or neptune-python-utils for SigV4 signing. IAM auth is separate from VPC security — both apply simultaneously.

## Guardrail 5: Neptune Serverless Scales in NCUs
Neptune Serverless uses Neptune Capacity Units (NCUs) for scaling. Each NCU provides approximately 2 GiB of memory and associated compute. You configure minimum and maximum NCU values. Scaling is automatic but not instantaneous — brief pauses may occur during scale-up events. Cost is based on NCUs consumed per second. Not all instance features are available in serverless mode.

## Guardrail 6: No Cross-Region Read Replicas — Use Global Database
Neptune does not support cross-region read replicas like Aurora MySQL/PostgreSQL. For cross-region disaster recovery or low-latency reads in other regions, use Neptune Global Database. Global database replicates the entire storage volume to secondary regions with typical replication lag under 1 second. Failover to a secondary region requires promoting it to a standalone cluster.

## Guardrail 7: Storage Auto-Scales Up to 128 TiB
Neptune uses a distributed storage layer similar to Aurora that automatically grows as data is added, up to 128 TiB. Storage cannot be manually sized or pre-allocated. You are billed for storage consumed, not provisioned. Storage does not automatically shrink when data is deleted — use the cluster cloning feature to reclaim space if needed.

## Guardrail 8: Instance Class Determines Query Performance
Neptune query performance is heavily dependent on instance class. The instance must have enough memory to hold the working set of the graph. If the graph exceeds instance memory, queries will be slower due to page cache misses. Use db.r5 or db.r6g instances for production workloads. db.t3/t4g instances are for development only and have CPU credit limitations.

## Guardrail 9: Query Timeout Defaults to 120 Seconds
Neptune has a default query timeout of 120 seconds (2 minutes). Long-running queries will be terminated. This can be adjusted via the neptune_query_timeout parameter in the DB cluster parameter group. For bulk analytical queries, increase the timeout. For OLTP workloads, keep it low to prevent runaway queries from consuming resources.

## Guardrail 10: Neptune Notebooks Connect Directly to Cluster
Neptune Workbench (Jupyter notebooks) connects directly to the Neptune cluster endpoint within the VPC. Notebooks must be in the same VPC or have network connectivity to the cluster. They support both Gremlin and SPARQL with built-in graph visualization. Notebooks use IAM roles for authentication — ensure the notebook execution role has neptune-db permissions.

## Guardrail 11: Engine Version Determines Feature Availability
Neptune engine versions control which features are available. Features like Neptune Serverless, Global Database, and certain query optimizations require specific minimum engine versions. Always check the engine version before troubleshooting missing features. Engine version upgrades can be applied during maintenance windows and may cause brief downtime.

## Guardrail 12: Parameter Group Changes May Require Reboot
Neptune cluster parameter groups control engine behavior. Some parameters are dynamic (apply immediately) and others are static (require instance reboot). The neptune_query_timeout parameter is dynamic. The neptune_enable_audit_log parameter requires reboot. Always check the apply type before modifying parameters and plan for potential downtime.
