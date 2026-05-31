# Amazon MemoryDB Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon MemoryDB issue.

## Guardrail 1: MemoryDB Is Durable — ElastiCache Is Not
MemoryDB writes to a durable multi-AZ transaction log before acknowledging writes. Data survives node failures, restarts, and failovers. ElastiCache is a cache — data may be lost on node failure (unless using Redis with Multi-AZ and backup). Never confuse the durability guarantees between the two services. MemoryDB is a primary database; ElastiCache is a cache layer.

## Guardrail 2: ACL Is the Only Authentication Method
MemoryDB uses Access Control Lists (ACLs) with username/password authentication. There is NO AUTH token option like ElastiCache. Each user has a username, password, and access string defining allowed commands and key patterns. The default user provides open access and should be restricted in production. Never suggest AUTH token configuration for MemoryDB.

## Guardrail 3: Cluster Mode Is Always Enabled
MemoryDB always operates in cluster mode with hash slot distribution (0-16383). There is no non-cluster mode option. Even single-shard clusters use the cluster protocol. Applications MUST use Redis Cluster-aware clients (e.g., redis-py-cluster, JedisCluster, ioredis with cluster mode). Standard Redis clients will fail with MOVED/ASK errors.

## Guardrail 4: TLS Is Always Required
MemoryDB requires TLS for all connections — it cannot be disabled. All clients must support TLS 1.2+. Use redis-cli with --tls flag. Connection strings must specify TLS. This differs from ElastiCache where TLS is optional and can be enabled/disabled. Never suggest connecting to MemoryDB without TLS.

## Guardrail 5: Transaction Log Adds Write Latency
MemoryDB's durability comes from writing to the transaction log before acknowledging writes. This adds latency compared to ElastiCache (typically 1-5ms additional). Read latency is comparable to ElastiCache since reads are served from memory. If write latency is critical and durability is not needed, ElastiCache may be more appropriate.

## Guardrail 6: Snapshot Cross-Service Compatibility Is One-Way
MemoryDB snapshots can be used to create ElastiCache clusters (MemoryDB → ElastiCache migration). ElastiCache snapshots CANNOT be used to create MemoryDB clusters. Migration from ElastiCache to MemoryDB requires using the MemoryDB snapshot of an ElastiCache cluster feature or online migration. Never suggest restoring an ElastiCache snapshot directly to MemoryDB.

## Guardrail 7: Node Type Determines Performance and Connection Limits
Each MemoryDB node type has specific CPU, memory, network bandwidth, and connection limits. Choosing an undersized node type is the most common cause of performance issues. Memory usage should stay below 80% to avoid eviction and OOM. Network bandwidth limits can cause throughput bottlenecks on smaller node types.

## Guardrail 8: Shard and Replica Limits
MemoryDB supports up to 500 shards per cluster and up to 5 replicas per shard. More shards increase write throughput and total memory. More replicas increase read throughput and availability. Resharding (adding/removing shards) is an online operation but can take time for large datasets. Replica count changes require cluster modification.

## Guardrail 9: Connection Limits Are Per-Node
Each node has a maximum connection limit based on node type. Connection limits are per-node, not per-cluster. The cluster endpoint distributes connections across shards. Monitor CurrConnections per node. Connection exhaustion on a single node can cause failures even if other nodes have capacity.

## Guardrail 10: Parameter Group Changes May Require Restart
MemoryDB parameter groups control engine behavior. Some parameter changes apply immediately (dynamic), while others require a node restart (static). Changes to maxmemory-policy apply immediately. Changes to cluster-related parameters may require restart. Always check parameter apply type before modifying.

## Guardrail 11: Eviction Policy Affects Data Durability
Even though MemoryDB is durable, the eviction policy (maxmemory-policy) determines what happens when memory is full. With volatile-lru or allkeys-lru, data IS evicted when memory is full — evicted data is removed from the transaction log too. Use noeviction if data loss is unacceptable (writes will fail instead). The default is volatile-lru.

## Guardrail 12: Multi-AZ Is Always Enabled
MemoryDB clusters are always deployed across multiple AZs for durability. The transaction log is replicated across AZs. Failover to a replica in a different AZ is automatic. You cannot create a single-AZ MemoryDB cluster. This ensures data durability even during AZ failures.
