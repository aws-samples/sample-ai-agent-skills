# Amazon MQ Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon MQ issue.

## Guardrail 1: ActiveMQ and RabbitMQ Are Different Engines
Amazon MQ supports two engines: ActiveMQ and RabbitMQ. They have different protocols (OpenWire/AMQP/STOMP/MQTT vs AMQP 0-9-1), different configuration formats, different monitoring metrics, and different troubleshooting approaches. Never apply ActiveMQ-specific advice to RabbitMQ brokers or vice versa. Always check the engine type first.

## Guardrail 2: Deployment Mode Cannot Be Changed After Creation
Single-instance, active/standby (ActiveMQ), and cluster (RabbitMQ) deployment modes are set at creation and cannot be changed. To change deployment mode, you must create a new broker and migrate. Active/standby provides automatic failover for ActiveMQ. RabbitMQ cluster provides HA with 3 nodes. Single-instance has no failover.

## Guardrail 3: Storage Full Causes Message Rejection
When StorePercentUsage reaches 100%, the broker stops accepting new messages. For ActiveMQ, this is the KahaDB store limit. For RabbitMQ, this is disk space. Monitor StorePercentUsage and set alarms at 80%. Purging queues or increasing storage (if EBS) can resolve the issue. Persistent messages consume more storage than non-persistent.

## Guardrail 4: VPC Security Groups Must Allow Broker Ports
ActiveMQ uses ports: 61617 (OpenWire+TLS), 5671 (AMQP+TLS), 61614 (STOMP+TLS), 8883 (MQTT+TLS), 8162 (Web Console). RabbitMQ uses ports: 5671 (AMQP+TLS), 443 (Management UI). Security groups must allow inbound traffic on these ports from client CIDRs. Missing rules cause connection timeouts, not explicit errors.

## Guardrail 5: Maintenance Windows Apply Updates Automatically
Amazon MQ applies minor version updates and patches during the maintenance window. This causes brief downtime for single-instance brokers. Active/standby brokers failover during maintenance. Major version upgrades must be initiated manually. You can defer maintenance but not indefinitely. Check pending maintenance with describe-broker.

## Guardrail 6: Configuration Changes Require Broker Reboot
Most broker configuration changes (XML for ActiveMQ, Cuttlefish for RabbitMQ) are not applied until the broker is rebooted. Reboot causes brief downtime. Some changes require creating a new configuration revision first. Always verify changes took effect after reboot. Use describe-configuration-revision to check pending changes.

## Guardrail 7: Instance Type Determines Resource Limits
Each instance type has specific CPU, memory, and network limits. mq.t3.micro is for development only (limited throughput). mq.m5.large and above are for production. Heap usage above 80% causes GC pressure and performance degradation. CPU above 80% sustained indicates need for larger instance or optimization.

## Guardrail 8: ActiveMQ Network of Brokers Has Specific Requirements
Network connectors must be configured between brokers. Duplex=true creates bidirectional connections. Message forwarding adds latency. Advisory messages can cause storms in large networks. Each broker needs the other's endpoint in its network connector configuration. Conditional forwarding can reduce unnecessary message movement.

## Guardrail 9: RabbitMQ Quorum Queues Require Cluster Deployment
Quorum queues provide data safety through Raft consensus and require a cluster deployment (3 nodes). They replace classic mirrored queues. Quorum queues have different performance characteristics — higher latency but better data safety. They do not support all classic queue features (e.g., message TTL per message, queue length limits behave differently).

## Guardrail 10: Message Persistence Affects Performance and Durability
Persistent messages survive broker restart but are slower (written to disk). Non-persistent messages are faster but lost on restart. ActiveMQ defaults to persistent. RabbitMQ defaults to non-persistent (delivery_mode=1). For high throughput with durability, use persistent messages with appropriate storage type.

## Guardrail 11: Protocol Selection Affects Features and Performance
ActiveMQ supports OpenWire (best performance), AMQP 1.0, STOMP, MQTT, and WebSocket. RabbitMQ supports AMQP 0-9-1 only on Amazon MQ. Each protocol has different features and performance characteristics. OpenWire is recommended for ActiveMQ Java clients. AMQP 0-9-1 is the only option for RabbitMQ.

## Guardrail 12: Broker State Transitions Indicate Lifecycle Stage
Broker states: CREATION_IN_PROGRESS → RUNNING → REBOOT_IN_PROGRESS → RUNNING. DELETION_IN_PROGRESS is terminal. CREATION_FAILED indicates a creation error. Only RUNNING brokers accept connections. State transitions take time — do not assume instant state changes. Check BrokerState in describe-broker output.
