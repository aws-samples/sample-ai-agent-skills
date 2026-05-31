---
name: mq-diagnostics
description: >
  Use this skill to investigate and troubleshoot Amazon MQ problems
  (ActiveMQ and RabbitMQ) by analyzing broker health, connectivity,
  performance, engine-specific issues, maintenance, and security using
  structured runbooks. Activate when: broker creation failures, broker
  health degraded, storage full, connection failures, TLS issues, VPC
  access problems, message throughput low, consumer lag, memory pressure,
  ActiveMQ-specific issues, network of brokers problems, RabbitMQ-specific
  issues, quorum queue problems, version upgrades, configuration changes,
  authentication failures, encryption issues, or the user says something
  is wrong with Amazon MQ without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with MQ, EC2, CloudWatch, CloudTrail,
  and optionally VPC and KMS permissions.
---

# Amazon MQ (ActiveMQ/RabbitMQ) Diagnostics

## When to use

Any Amazon MQ investigation where the console alone is insufficient — broker health, connectivity issues, performance tuning, engine-specific problems, maintenance operations, or security configuration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mq list-brokers --query 'BrokerSummaries[*].{Id:BrokerId,Name:BrokerName,Engine:EngineType,State:BrokerState}'
aws mq describe-broker --broker-id <broker-id>
aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name CpuUtilization --dimensions Name=Broker,Value=<broker-name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name StorePercentUsage --dimensions Name=Broker,Value=<broker-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
aws mq describe-broker --broker-id <broker-id> --query '{Engine:EngineType,Version:EngineVersion,DeploymentMode:DeploymentMode,HostInstanceType:HostInstanceType,Storage:StorageType}'
aws mq describe-configuration --configuration-id <config-id>
aws mq describe-configuration-revision --configuration-id <config-id> --configuration-revision <rev>
aws mq list-users --broker-id <broker-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mq.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name HeapUsage --dimensions Name=Broker,Value=<broker-name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws logs filter-log-events --log-group-name /aws/amazonmq/broker/<broker-id>/general --start-time <epoch>
```

Read `references/mq-guardrails.md` before concluding on any Amazon MQ issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-broker` | Check broker config, state, engine, deployment mode |
| `list-brokers` | List all brokers and their states |
| `describe-configuration` | Check broker configuration details |
| `list-users` | Check broker users |
| CloudWatch Metrics | Check CPU, memory, storage, queue depth |
| CloudWatch Logs | Check broker general and audit logs |
| Web Console | Access ActiveMQ/RabbitMQ management UI |

## Gotchas: Amazon MQ

- ActiveMQ and RabbitMQ are different engines with different behaviors. ActiveMQ uses OpenWire, AMQP, STOMP, MQTT, WSS protocols. RabbitMQ uses AMQP 0-9-1. Configuration, monitoring, and troubleshooting differ significantly between engines.
- Broker deployment mode affects availability. Single-instance has no failover. Active/standby (ActiveMQ) provides automatic failover. Cluster (RabbitMQ) provides high availability. Deployment mode cannot be changed after creation.
- Storage type affects performance and limits. EBS storage is the default. EFS is available for ActiveMQ active/standby. Storage limits vary by instance type. StorePercentUsage at 100% causes message rejection.
- VPC access requires proper security group and subnet configuration. Brokers in VPC need security groups allowing client ports (61617 for ActiveMQ, 5671 for RabbitMQ). Public accessibility can be enabled but is not recommended for production.
- Maintenance windows apply updates automatically. Minor version updates and patches are applied during the maintenance window. Major version upgrades require manual initiation. Maintenance causes brief downtime for single-instance brokers.
- Configuration changes require broker reboot. Most configuration changes are not applied until the broker is rebooted. Some changes require creating a new configuration revision. Reboot causes brief downtime.
- Memory and CPU limits depend on instance type. mq.m5.large has different limits than mq.t3.micro. Heap usage above 80% causes performance degradation. CPU above 80% indicates need for larger instance.
- ActiveMQ network of brokers has specific requirements. Network connectors must be configured on both sides. Duplex connections reduce configuration. Message forwarding has latency. Advisories can cause message storms.

## Anti-hallucination rules

1. Always cite specific broker IDs, metric values, or API responses as evidence.
2. ActiveMQ and RabbitMQ are different engines. Never apply ActiveMQ advice to RabbitMQ or vice versa.
3. Deployment mode cannot be changed after creation. Never suggest changing it.
4. Storage full causes message rejection. Never ignore StorePercentUsage warnings.
5. Configuration changes require reboot. Never assume changes apply immediately.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Broker | A1-A3 | Creation failures, broker health, storage full |
| B — Connectivity | B1-B3 | Connection failures, TLS, VPC access |
| C — Performance | C1-C3 | Message throughput, consumer lag, memory pressure |
| D — ActiveMQ | D1-D2 | ActiveMQ-specific issues, network of brokers |
| E — RabbitMQ | E1-E2 | RabbitMQ-specific issues, quorum queues |
| F — Maintenance | F1-F2 | Version upgrades, configuration changes |
| G — Security | G1-G2 | Authentication, encryption |
| Z — Catch-All | Z1 | General troubleshooting |
