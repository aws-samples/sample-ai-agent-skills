# Aurora Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Aurora issue.

## Guardrail 1: Storage Cannot Shrink (Only Grows)
Aurora storage auto-scales up to 128 TiB in 10 GiB increments. However, storage never shrinks even after data is deleted. The freed space is reused internally by the storage engine but the billed volume size does not decrease. To reclaim storage, create a new cluster from a snapshot or use logical dump and restore. Do not tell users that deleting data or dropping tables will reduce their storage bill.

## Guardrail 2: Reader Endpoint Is DNS Round-Robin (Not a Load Balancer)
The Aurora reader endpoint distributes connections via DNS round-robin across available reader instances. It does NOT provide connection-level or query-level load balancing. Successive DNS lookups may resolve to the same reader. For true load balancing, recommend RDS Proxy or application-level connection routing with individual instance endpoints.

## Guardrail 3: Writer Failover Takes ~30 Seconds (DNS TTL)
Aurora writer failover typically completes in ~30 seconds when a reader exists to be promoted. The cluster endpoint DNS TTL is 5 seconds. Applications must handle DNS caching (disable JVM DNS cache or set short TTL), implement connection retry logic, and be prepared for brief unavailability. If no reader exists, failover takes longer because a new instance must be created.

## Guardrail 4: Backtrack Is MySQL Only
Aurora backtrack (rewind to a point in time without restore) is available only on Aurora MySQL. It is NOT available on Aurora PostgreSQL. Do not suggest backtrack for PostgreSQL clusters. For PostgreSQL, point-in-time restore (creating a new cluster) is the only option.

## Guardrail 5: Serverless v2 Scaling Has Latency
Aurora Serverless v2 does not scale instantaneously. Scaling up from minimum ACU takes time and there can be noticeable latency during scale-up events, especially from very low ACU values. Cold start occurs when the cluster scales from near-zero activity. Recommend setting minimum ACU high enough to handle baseline traffic without scaling delays.

## Guardrail 6: Global Database RPO/RTO Depends on Failover Type
Aurora Global Database has two failover mechanisms with different characteristics. Managed planned failover: RPO = 0 (no data loss), RTO < 1 minute, maintains replication topology. Unplanned failover (detach and promote): RPO typically < 1 second but data loss is possible, requires manual replication re-setup. Do not conflate these two operations or claim zero data loss for unplanned failover.

## Guardrail 7: Cloning Is Copy-on-Write (Not an Instant Full Copy)
Aurora cloning uses a copy-on-write protocol. Clone creation is near-instant regardless of database size because it shares storage pages with the source. However, as pages are modified on either the source or clone, new pages are allocated. The clone is a full independent cluster — not a snapshot or read replica. Storage costs increase as the clone diverges from the source.

## Guardrail 8: Aurora I/O-Optimized Eliminates I/O Charges but Higher Instance Cost
Aurora I/O-Optimized pricing eliminates per-I/O charges but increases instance cost by approximately 30%. It is cost-effective for I/O-heavy workloads where I/O charges exceed 25% of total Aurora cost. You can switch between Standard and I/O-Optimized once every 30 days. Do not recommend I/O-Optimized for read-light workloads without cost analysis.

## Guardrail 9: Parallel Query Is MySQL Only
Aurora parallel query pushes query processing to the storage layer for analytical queries. It is available only on Aurora MySQL with specific engine versions and instance classes. It is NOT available on Aurora PostgreSQL. Do not suggest parallel query for PostgreSQL clusters.

## Guardrail 10: Cluster vs Instance Parameter Groups
Aurora uses two levels of parameter groups. Cluster parameter groups apply to all instances in the cluster and control engine-level settings (e.g., binlog_format, rds.logical_replication). Instance parameter groups apply to individual instances and control instance-specific settings. Some parameters exist only at one level. Changing cluster parameters may require a cluster reboot; changing instance parameters may require an instance reboot. Do not confuse the two levels.

## Guardrail 11: Storage Replication Is Synchronous Within Region
Aurora replicates data synchronously across 6 copies in 3 AZs within a region. This provides durability but means write operations include replication latency. Aurora can tolerate the loss of 2 copies for writes and 3 copies for reads without data loss. Cross-region replication (Global Database) is asynchronous with typical lag < 1 second.

## Guardrail 12: Local Storage for Temp Tables Is Instance-Specific
Each Aurora instance has local NVMe-based storage used for temporary tables, sort operations, and join buffers. This storage is NOT shared across the cluster and is NOT part of the Aurora shared storage volume. Local storage size depends on the instance class. If an instance runs out of local storage, queries using temp tables will fail. Scaling up the instance class increases local storage.
