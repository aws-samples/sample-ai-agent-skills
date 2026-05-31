# Amazon DocumentDB Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon DocumentDB issue.

## Guardrail 1: DocumentDB Is NOT MongoDB
DocumentDB is API-compatible with MongoDB but uses a completely different storage engine (Aurora-based). Not all MongoDB features, operators, or commands are supported. Always verify feature support in the DocumentDB documentation, not MongoDB documentation. Unsupported features include: $graphLookup (limited), certain aggregation stages, capped collections, and some index types.

## Guardrail 2: TLS Is Required by Default
All DocumentDB connections require TLS by default. The rds-combined-ca-bundle.pem or region-specific certificate must be specified in the connection string. Disabling TLS requires setting the tls parameter to "disabled" in the cluster parameter group. Never recommend disabling TLS in production. Connection strings must include the tlsCAFile parameter.

## Guardrail 3: Change Streams Have Limited Retention
Change stream events are retained for 24 hours by default. This can be extended up to 7 days by modifying the change_stream_log_retention_duration cluster parameter. Events older than the retention period are permanently deleted. Consumers must maintain resume tokens and process events within the retention window. Lost resume tokens require re-reading from the current position.

## Guardrail 4: Index Builds Block Writes
DocumentDB index creation is a foreground operation that blocks write operations on the collection. Unlike MongoDB 4.2+ which supports background index builds, DocumentDB does not have true background index building. Plan index creation during maintenance windows or low-traffic periods. Monitor index build progress with db.currentOp(). Large collections may take significant time to index.

## Guardrail 5: Elastic Clusters Are Different from Standard Clusters
Elastic clusters use sharding to distribute data across multiple shards. They have different connection endpoints, API operations, and feature support compared to standard clusters. Not all standard cluster features work in elastic clusters. Elastic clusters require a shard key for data distribution. Migration between standard and elastic clusters requires data export/import.

## Guardrail 6: Parameter Groups Are Not MongoDB Config Files
DocumentDB uses cluster parameter groups (similar to RDS) to configure engine behavior. Parameters like profiler threshold, TTL monitor, audit logging, and TLS are controlled via parameter groups. Some MongoDB configuration options have no DocumentDB equivalent. Parameter changes may require cluster restart depending on the parameter type (static vs dynamic).

## Guardrail 7: No mongodump/mongorestore for Backup
DocumentDB does not support mongodump/mongorestore for backup and restore. Use AWS native snapshots for point-in-time backups, PITR for continuous backup (up to 35 days), or AWS DMS for migration. For logical data export, use mongoexport/mongoimport or custom application-level scripts. Snapshots are cluster-level, not database or collection-level.

## Guardrail 8: Aggregation Pipeline Has Limitations
Not all MongoDB aggregation pipeline stages are supported in DocumentDB. Support varies by engine version (3.6, 4.0, 5.0). $lookup support was added in 4.0 with limitations and expanded in 5.0. $merge and $out have version-specific support. Always check the DocumentDB aggregation support matrix for your engine version before building pipelines.

## Guardrail 9: Storage Auto-Scales but Cannot Be Manually Shrunk
DocumentDB storage automatically grows as data is added, up to 128 TiB (standard) or 4 PiB (elastic). Storage is billed based on high-water mark — deleting data does not immediately reduce the storage bill. Storage reclamation happens gradually over time. There is no manual shrink operation. To reclaim storage immediately, you must dump data and restore to a new cluster.

## Guardrail 10: Instance Class Determines Performance Limits
Each DocumentDB instance class has specific limits for CPU, memory, network bandwidth, and maximum connections. Choosing an undersized instance class is the most common cause of performance issues. The writer instance handles all writes; reader instances handle read traffic. Scale readers horizontally for read-heavy workloads. Scale the writer instance class vertically for write-heavy workloads.

## Guardrail 11: Connection Limits Are Per-Instance
Each DocumentDB instance has a maximum connection limit determined by its instance class (e.g., db.r5.large supports ~1,700 connections). Connection limits are per-instance, not per-cluster. The cluster endpoint routes to the writer; the reader endpoint load-balances across readers. Connection pooling is essential — each application instance should maintain a pool, not open connections per request.

## Guardrail 12: Engine Version Determines Feature Availability
DocumentDB engine versions (3.6, 4.0, 5.0) have significantly different feature sets. Features like $lookup, change streams enhancements, ACID transactions, and aggregation stages vary by version. Upgrading engine versions is a cluster modification that causes brief downtime. Always test application compatibility before upgrading. Downgrading engine versions is not supported.
