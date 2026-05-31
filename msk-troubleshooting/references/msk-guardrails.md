# Amazon MSK Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any MSK issue.

## Guardrail 1: Broker Storage Is NOT Auto-Scaling by Default
MSK provisioned clusters do not automatically scale broker storage unless you explicitly enable auto-scaling. Without it, brokers run out of disk and become unhealthy when `KafkaDataLogsDiskUsed` reaches 100%. Enable auto-scaling via the console, CLI (`aws kafka update-storage`), or API. Auto-scaling has a cooldown period and scales in 10 GiB increments up to the configured maximum. Never assume storage will grow automatically on a new or existing cluster without verifying auto-scaling is enabled.

## Guardrail 2: Replication Factor vs min.insync.replicas — They Are Different Settings
The replication factor (set per topic at creation) determines how many copies of each partition exist across brokers. `min.insync.replicas` (broker or topic config) determines how many replicas must acknowledge a write when `acks=all`. If `min.insync.replicas` equals the replication factor, a single broker failure blocks all writes to that topic. The recommended production setting is replication factor = 3 and `min.insync.replicas` = 2. Never set `min.insync.replicas` >= replication factor in production.

## Guardrail 3: Consumer Group Lag Is the Primary Health Indicator
Consumer lag (difference between latest offset and consumer's committed offset) is the most important metric for consumer health. High lag means consumers cannot keep up with producers. Check `MaxOffsetLag`, `SumOffsetLag`, and `EstimatedMaxTimeLag` CloudWatch metrics per consumer group. Lag can be caused by slow processing, insufficient consumer instances, frequent rebalancing, or producer bursts. Never dismiss high consumer lag as normal without investigating the cause.

## Guardrail 4: Partition Count Can Only Increase, Never Decrease
Once a topic is created with N partitions, you can only increase the count, never decrease it. Increasing partitions changes key-based message ordering guarantees because the key-to-partition mapping changes. Plan partition count carefully based on expected throughput. Deleting and recreating a topic to reduce partitions causes data loss. Never suggest reducing partition count — recommend creating a new topic and migrating consumers instead.

## Guardrail 5: MSK Connect Uses Custom Plugins from S3 — Not Direct JAR Uploads
MSK Connect requires connector plugins to be uploaded to S3 as ZIP or JAR files, then registered as custom plugins. You cannot directly upload JARs to MSK Connect. Not all community Kafka Connect connectors work without modification — some require specific dependencies or configurations. MSK Connect workers have different resource limits than self-managed Connect clusters. Never assume a self-managed connector configuration will work identically on MSK Connect.

## Guardrail 6: Authentication Ports Are Method-Specific — Do Not Mix
Each MSK authentication method uses a specific port: IAM auth uses port 9098 (TLS), SASL/SCRAM uses port 9096 (TLS), mTLS uses port 9094 (TLS), and plaintext (unauthenticated) uses port 9092. Using the wrong port for an auth method causes connection failures. The `get-bootstrap-brokers` API returns separate connection strings for each enabled auth method. A cluster can support multiple auth methods simultaneously, but each client must connect to the correct port. Never mix auth method ports.

## Guardrail 7: MSK Serverless Has Strict Limits Different from Provisioned
MSK Serverless automatically manages infrastructure but has hard limits: max 5 consumer groups reading from the same partition simultaneously, max 120 MB/s ingress and 360 MB/s egress per cluster, max 200 partitions per topic, max 50 topics per cluster. Serverless supports IAM authentication only — no SASL/SCRAM or mTLS. No custom broker configurations. No multi-VPC connectivity. Never apply provisioned cluster troubleshooting (broker instance types, custom configs, storage scaling) to Serverless clusters.

## Guardrail 8: Topic-Level Configs Override Broker-Level Defaults
Topic-level configurations (retention.ms, max.message.bytes, cleanup.policy, compression.type, etc.) override the corresponding broker-level defaults. When troubleshooting retention, message size, or compaction issues, always check BOTH topic-level overrides and broker-level defaults. Use `kafka-configs.sh --describe --entity-type topics --entity-name <topic>` to see topic-level overrides. A topic with no overrides inherits all broker defaults. Never assume broker-level settings apply if topic-level overrides exist.

## Guardrail 9: Under-Replicated Partitions Signal Broker Health Problems
The `UnderReplicatedPartitions` metric is a critical health indicator. Non-zero values mean some partition replicas are not in sync with the leader. Causes include: broker CPU/disk/network overload, broker failure, uneven partition distribution, large messages, or insufficient replication throughput. Sustained under-replication risks data loss if the leader broker fails before replicas catch up. Never ignore non-zero `UnderReplicatedPartitions` — investigate immediately.

## Guardrail 10: Storage Scaling Is One-Way — Cannot Shrink Broker Volumes
Once broker EBS volume size is increased (manually or via auto-scaling), it cannot be decreased. This applies to both manual scaling and auto-scaling. Plan initial storage carefully. Use topic retention policies (`retention.ms`, `retention.bytes`) and log compaction (`cleanup.policy=compact`) to manage data growth. The minimum storage per broker is 1 GiB and maximum is 16384 GiB (16 TiB). Never suggest decreasing broker storage volume size.

## Guardrail 11: Bootstrap Brokers Are for Initial Connection Only
Bootstrap broker endpoints are used for initial client connection and metadata discovery. Clients then connect directly to individual brokers for producing/consuming. If security groups or network ACLs block direct broker-to-broker or client-to-broker communication on the required ports, clients will fail after initial connection. Ensure all broker ports (9092, 9094, 9096, 9098) and inter-broker port (9093) are open in security groups. Never assume only the bootstrap port needs to be open.

## Guardrail 12: Cluster Configuration Updates Cause Rolling Restarts
Updating MSK cluster configuration (broker config, instance type, Kafka version, security settings) triggers a rolling restart of brokers. During rolling restarts, one broker at a time is restarted, causing temporary under-replication and potential producer/consumer disruption. Updates can take 15-30+ minutes depending on cluster size. Configuration updates cannot be cancelled once started. Never apply configuration changes during peak traffic without understanding the rolling restart impact.
