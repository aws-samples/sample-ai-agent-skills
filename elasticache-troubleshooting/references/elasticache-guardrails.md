# Amazon ElastiCache Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any ElastiCache issue.

## Guardrail 1: maxmemory-reserved — Always Account for Non-Data Overhead
AWS recommends reserving 25% of maxmemory for non-data overhead including replication output buffers, Lua script memory, pub/sub output buffers, and connection overhead. Insufficient reservation causes OOM during replication sync or heavy write loads even when used_memory appears below maxmemory. Check the parameter group value — it is set per node type with AWS defaults. Never assume all maxmemory is available for data.

## Guardrail 2: Eviction Policy Selection — volatile-* Only Evicts Keys with TTL
volatile-lru, volatile-lfu, volatile-random, and volatile-ttl only evict keys that have an expiration (TTL) set. If no keys have TTL and the policy is volatile-*, Redis returns OOM errors instead of evicting. For cache-only workloads, use allkeys-lru or allkeys-lfu. For mixed workloads (cache + persistent data), use volatile-lru with TTL on cache keys. Never assume volatile-* will evict all keys.

## Guardrail 3: In-Transit Encryption (TLS) — Creation-Time Only
TLS can only be enabled when creating a new cluster or replication group. It cannot be added to an existing non-TLS cluster. Migrating to TLS requires creating a new cluster with TLS enabled and migrating data. TLS adds approximately 25% CPU overhead. When TLS is enabled, all clients must use TLS connections — redis-cli requires the --tls flag. Never suggest enabling TLS on an existing cluster without recreation.

## Guardrail 4: Memcached Has No Replication, Persistence, or Backups
Memcached is purely in-memory with no persistence, no replication, no failover, no backups, and no snapshots. Data is lost when nodes are removed, replaced, or restarted. Memcached scales horizontally by adding nodes — data is distributed via client-side consistent hashing. Never suggest replication, backups, failover, or persistence features for Memcached clusters.

## Guardrail 5: Cluster Mode Enabled vs Disabled — Cannot Convert In-Place
Cluster mode enabled (CME) and cluster mode disabled (CMD) are fundamentally different architectures. CMD has a single shard (one primary, up to 5 replicas). CME has multiple shards (up to 500 nodes total). You cannot convert CMD to CME or vice versa in-place — you must create a new cluster and migrate data. CME requires Redis Cluster-aware clients. Multi-key commands in CME must operate on keys in the same hash slot.

## Guardrail 6: Failover DNS Propagation — 30-60 Seconds, Not Instant
When Multi-AZ automatic failover promotes a replica to primary, the primary endpoint DNS record is updated. DNS propagation takes 30-60 seconds. Applications may experience brief connection errors during this window. Applications should implement retry logic with exponential backoff. The reader endpoint continues to work during failover. Never claim failover is instant or seamless.

## Guardrail 7: Connection Limits Vary by Node Type
Each node type has a maximum connection limit (maxclients). Smaller node types (cache.t3.micro, cache.t3.small) have lower limits. Monitor CurrConnections in CloudWatch. Connection storms from application restarts or connection pool misconfiguration can exhaust limits. New connections are refused when the limit is reached. Always check the node type's maxclients value before diagnosing connection issues.

## Guardrail 8: Parameter Group Changes — Some Require Reboot
Not all parameter group changes apply immediately. Changes to maxmemory-policy, timeout, and tcp-keepalive apply immediately. Changes to maxmemory-reserved, cluster-enabled, and appendonly require a reboot. Check the "Apply Type" column (immediate vs requires-reboot) in describe-cache-parameters output. Never assume all parameter changes take effect without a reboot.

## Guardrail 9: Redis Backup Uses fork() — Temporarily Doubles Memory
Redis snapshots use fork() to create a child process for background saving. This temporarily requires up to 2x the memory of the dataset due to copy-on-write. If the node does not have sufficient free memory, the backup fails or causes OOM. Schedule backups during low-traffic periods. Ensure the node type has enough memory headroom. Memcached does not support backups at all.

## Guardrail 10: Global Datastore Failover Is Manual, Not Automatic
Global Datastore provides cross-region replication but failover to a secondary region is a manual operation, not automatic. The secondary region is read-only until promoted. Promotion requires calling failover-global-replication-group. Typical replication lag is under 1 second but can increase during heavy write loads. Requires Redis 5.0.6+ and cluster mode enabled.

## Guardrail 11: ElastiCache Serverless — Different Limits and Pricing Model
ElastiCache Serverless automatically scales but has different limits than provisioned clusters. Maximum item size, connection limits, and supported commands may differ. Pricing is based on data stored (GB-hours) and ElastiCache Processing Units (ECPUs), not node hours. Not all Redis commands are supported. Check AWS documentation for current Serverless limits before recommending Serverless.

## Guardrail 12: Clusters Are VPC-Only — Not Publicly Accessible
ElastiCache clusters run inside a VPC and are not publicly accessible. The subnet group defines which subnets and AZs the cluster uses. Security groups control inbound access on port 6379 (Redis) or 11211 (Memcached). To connect from outside the VPC, use a bastion host, VPN, VPC peering, or Transit Gateway. Never suggest direct internet access to ElastiCache nodes.
