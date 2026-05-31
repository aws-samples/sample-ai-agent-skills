# Amazon Keyspaces Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Keyspaces issue.

## Guardrail 1: Keyspaces Is NOT Apache Cassandra
Amazon Keyspaces is a serverless, CQL-compatible database service. It is NOT open-source Cassandra. Do not use Cassandra-native tools (nodetool, cassandra-stress, sstableloader) or suggest Cassandra-specific configurations (cassandra.yaml, commitlog, compaction strategies). The underlying storage engine is completely different. Cassandra documentation does not apply directly.

## Guardrail 2: TLS Is Mandatory for All Connections
Keyspaces requires TLS encryption on all connections. The service listens on port 9142 (not Cassandra's default 9042). Clients must trust the Starfield digital certificate (or the Amazon root CA). Connections without TLS are rejected. The certificate must be downloaded and configured in the client's SSL settings.

## Guardrail 3: Authentication Uses SigV4 or Service-Specific Credentials Only
Keyspaces does not support native Cassandra authentication (PasswordAuthenticator). Use IAM-based SigV4 authentication (recommended) via the SigV4 auth plugin for cqlsh/drivers, or generate service-specific credentials (username/password) through the IAM console. Service-specific credentials are tied to an IAM user and inherit that user's permissions.

## Guardrail 4: Partition Key Design Is Critical — Same Principles as DynamoDB
Keyspaces distributes data across partitions based on the partition key hash. A single hot partition can throttle the entire table even if overall capacity is available. Design partition keys for even distribution. Avoid monotonically increasing keys (timestamps, sequential IDs) as sole partition keys. Use composite partition keys for high-cardinality distribution.

## Guardrail 5: CQL Compatibility Is Partial — Not All CQL Features Are Supported
Keyspaces supports Apache Cassandra Query Language (CQL) 3.x with limitations. Unsupported features include: ALLOW FILTERING (restricted), lightweight transactions (limited IF support), user-defined types (UDTs) have limitations, BATCH statements limited to single partition, no TRUNCATE, no ALTER TYPE. Always verify feature support against current Keyspaces documentation.

## Guardrail 6: No UDFs, UDAs, Materialized Views, or Triggers
User-defined functions (UDFs), user-defined aggregates (UDAs), materialized views, and triggers are not supported. These are common Cassandra features that do not exist in Keyspaces. Suggest application-side logic, secondary indexes, or separate query tables as alternatives. Do not suggest creating materialized views for query optimization.

## Guardrail 7: TTL Maximum Is 630,720,000 Seconds (20 Years)
Time-to-live values must be between 1 and 630,720,000 seconds. Values exceeding this maximum are rejected with a validation error. TTL is set per-row using INSERT or UPDATE statements with the USING TTL clause. TTL can also be set as a default on the table. Expired data is deleted asynchronously — rows may be visible briefly after TTL expiration. TTL expiration deletes consume write capacity.

## Guardrail 8: PITR Is Continuous Backup with 35-Day Retention
Point-in-time recovery provides continuous backups with a 35-day retention window. PITR must be enabled on the table (it is not enabled by default). Restores always create a NEW table — you cannot restore in-place. Restore time depends on table size. PITR does not replace the need for cross-region backup strategies. There are no manual snapshot commands like Cassandra's nodetool snapshot.

## Guardrail 9: On-Demand Mode Has Initial Throughput Limits
On-demand capacity mode automatically scales but has initial limits. A new on-demand table starts with a previous peak of 4,000 write capacity units and 12,000 read capacity units. Traffic exceeding double the previous peak may be throttled temporarily. Switching from provisioned to on-demand takes effect immediately but can only be done once every 24 hours.

## Guardrail 10: Auto-Scaling for Provisioned Mode Uses Application Auto Scaling
Provisioned tables use AWS Application Auto Scaling (not built-in). You must configure scalable targets and scaling policies separately for read and write capacity. Auto-scaling reacts to CloudWatch alarms and has a cooldown period. It does NOT handle sudden spikes — use on-demand mode for unpredictable workloads. Target utilization is typically set to 70%.

## Guardrail 11: ALLOW FILTERING Queries Are Restricted
Unlike open-source Cassandra, Keyspaces restricts ALLOW FILTERING queries. Queries should use the partition key for efficient access. Full table scans are expensive and may time out. Design your data model around query patterns. Use secondary indexes (with limitations) for non-partition-key queries. Scans without partition key filters consume significant read capacity.

## Guardrail 12: No Multi-Region Tables — Single Region Only
Keyspaces tables exist in a single AWS region. There is no built-in multi-region replication like Cassandra's multi-datacenter replication or DynamoDB Global Tables. For multi-region requirements, implement application-level replication, use AWS DMS, or consider DynamoDB Global Tables as an alternative. Cross-region backup requires manual export/import or PITR restore to another region.
