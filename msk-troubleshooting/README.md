# MSK Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon MSK (Managed Streaming for Apache Kafka) and MSK Serverless problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon MSK when the console alone isn't enough — cluster health debugging, broker failure analysis, producer/consumer performance tuning, connectivity troubleshooting, authentication resolution, topic management, MSK Connect issues, security configuration, monitoring setup, and Serverless capacity problems.

### Activate When

- Cluster creation failures
- Broker failures or unhealthy brokers
- Storage full or approaching capacity
- Producer throughput degradation
- Consumer lag (high offset lag)
- High end-to-end latency
- Under-replicated partitions
- Client connection failures (timeouts, refused)
- Authentication errors (IAM, SASL/SCRAM, mTLS)
- VPC or security group connectivity issues
- Topic creation errors
- Partition rebalancing problems
- Retention or data loss issues
- MSK Connect connector failures
- MSK Connect connector scaling issues
- IAM authentication configuration problems
- SASL/SCRAM secret or auth failures
- TLS/mTLS certificate issues
- Missing CloudWatch metrics
- Open monitoring (Prometheus) setup issues
- MSK Serverless cluster creation or access issues
- MSK Serverless capacity or throughput limits

---

## Skill Structure

```
msk-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cluster-creation-failures.md
    ├── A2-broker-failures.md
    ├── A3-storage-full.md
    ├── B1-producer-throughput.md
    ├── B2-consumer-lag.md
    ├── B3-high-latency.md
    ├── B4-under-replicated-partitions.md
    ├── C1-client-connection-failures.md
    ├── C2-authentication-issues.md
    ├── C3-vpc-security-group.md
    ├── D1-topic-creation-errors.md
    ├── D2-partition-rebalancing.md
    ├── D3-retention-issues.md
    ├── E1-connector-failures.md
    ├── E2-connector-scaling.md
    ├── F1-iam-auth.md
    ├── F2-sasl-scram.md
    ├── F3-tls-mtls.md
    ├── G1-missing-metrics.md
    ├── G2-open-monitoring.md
    ├── H1-serverless-cluster-issues.md
    ├── H2-serverless-capacity.md
    ├── Z1-general-troubleshooting.md
    ├── msk-guardrails.md
    └── msk-hallucination-patterns.yaml
```

---

## Runbook Library (28 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A3 | Creation failures, broker failures, storage full |
| **B — Performance** | B1–B4 | Producer throughput, consumer lag, high latency, under-replicated partitions |
| **C — Connectivity** | C1–C3 | Client connection failures, authentication issues, VPC/security group |
| **D — Topics** | D1–D3 | Topic creation errors, partition rebalancing, retention issues |
| **E — MSK Connect** | E1–E2 | Connector failures, connector scaling |
| **F — Security** | F1–F3 | IAM auth, SASL/SCRAM, TLS/mTLS |
| **G — Monitoring** | G1–G2 | Missing metrics, open monitoring (Prometheus) |
| **H — Serverless** | H1–H2 | Serverless cluster issues, capacity/throughput |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/msk-guardrails.md` covering broker storage auto-scaling behavior, replication factor vs min.insync.replicas, consumer group lag interpretation, partition count immutability, MSK Connect vs self-managed connectors, IAM vs SASL/SCRAM vs mTLS authentication ports, MSK Serverless limits, topic-level vs broker-level configs, under-replicated partitions meaning, storage scaling one-way constraint, bootstrap broker endpoint usage, and cluster configuration update behavior.

---

## Investigation Workflow

1. **Triage** — Collect cluster config, broker node status, key CloudWatch metrics (ActiveControllerCount, OfflinePartitionsCount, KafkaDataLogsDiskUsed)
2. **Deep Dive** — Examine cluster operations, bootstrap brokers, per-broker CPU/disk/network metrics, under-replicated partitions
3. **Detailed** — Review broker configuration, CloudTrail events, SCRAM secrets, Kafka CLI topic/consumer-group diagnostics

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `kafka:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `ec2:DescribeSecurityGroups`, `ec2:DescribeSubnets`, `ec2:DescribeVpcs`, `iam:GetRole`, `secretsmanager:ListSecrets`, `kms:DescribeKey`
- Apache Kafka CLI tools (kafka-topics.sh, kafka-consumer-groups.sh, kafka-configs.sh) for direct cluster diagnostics
- Network access to MSK broker endpoints (VPC access required — MSK has no public endpoints by default)
- Java runtime for Kafka CLI tools

---

## Usage Examples

```
# List all MSK clusters
aws kafka list-clusters-v2

# Describe a specific cluster
aws kafka describe-cluster-v2 --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/abc-123

# Get bootstrap broker endpoints
aws kafka get-bootstrap-brokers --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/abc-123

# List broker nodes
aws kafka list-nodes --cluster-arn arn:aws:kafka:us-east-1:123456789012:cluster/my-cluster/abc-123

# Check under-replicated partitions
aws cloudwatch get-metric-statistics --namespace AWS/Kafka \
  --metric-name UnderReplicatedPartitions --dimensions Name=Cluster\ Name,Value=my-cluster Name=Broker\ ID,Value=1 \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Maximum

# Check consumer group lag (using Kafka CLI)
kafka-consumer-groups.sh --bootstrap-server <broker>:9092 --describe --group <consumer-group>
```

---

## License

MIT-0
