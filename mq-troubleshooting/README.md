# Amazon MQ (ActiveMQ/RabbitMQ) Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon MQ problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon MQ when the console alone isn't enough — broker health, connectivity, performance tuning, ActiveMQ-specific issues, RabbitMQ-specific issues, network of brokers, quorum queues, maintenance operations, authentication, and encryption.

### Activate When

- Broker creation failures
- Broker health degraded or unreachable
- Storage full causing message rejection
- Client connection failures
- TLS/SSL connection issues
- VPC access problems
- Low message throughput
- Consumer lag increasing
- Memory pressure or heap usage high
- ActiveMQ-specific issues (advisories, KahaDB)
- Network of brokers problems
- RabbitMQ-specific issues (exchanges, bindings)
- Quorum queue problems
- Version upgrade failures
- Configuration changes not applying
- Authentication failures
- Encryption configuration issues
- General Amazon MQ errors without clear symptoms

---

## Skill Structure

```
mq-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-broker-health.md
    ├── A3-storage-full.md
    ├── B1-connection-failures.md
    ├── B2-tls-issues.md
    ├── B3-vpc-access.md
    ├── C1-message-throughput.md
    ├── C2-consumer-lag.md
    ├── C3-memory-pressure.md
    ├── D1-activemq-specific.md
    ├── D2-network-of-brokers.md
    ├── E1-rabbitmq-specific.md
    ├── E2-quorum-queues.md
    ├── F1-version-upgrades.md
    ├── F2-configuration-changes.md
    ├── G1-authentication.md
    ├── G2-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── mq-guardrails.md
    └── mq-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Broker** | A1–A3 | Creation failures, broker health, storage full |
| **B — Connectivity** | B1–B3 | Connection failures, TLS, VPC access |
| **C — Performance** | C1–C3 | Message throughput, consumer lag, memory pressure |
| **D — ActiveMQ** | D1–D2 | ActiveMQ-specific issues, network of brokers |
| **E — RabbitMQ** | E1–E2 | RabbitMQ-specific issues, quorum queues |
| **F — Maintenance** | F1–F2 | Version upgrades, configuration changes |
| **G — Security** | G1–G2 | Authentication, encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/mq-guardrails.md` covering ActiveMQ vs RabbitMQ engine differences, deployment mode immutability, storage limits and behavior, VPC security group requirements, maintenance window behavior, configuration reboot requirement, instance type capacity limits, network of brokers requirements, quorum queue behavior, message persistence, protocol differences, and broker state transitions.

---

## Investigation Workflow

1. **Triage** — Check broker state, engine type, deployment mode, key metrics
2. **Deep Dive** — Examine configuration, users, logs, queue/topic details
3. **Detailed** — Review CloudTrail events, CloudWatch metrics trends, broker logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `mq:Describe*`, `mq:List*`, `cloudwatch:GetMetricStatistics`, `logs:FilterLogEvents`, `cloudtrail:LookupEvents`
- Access to broker web console (ActiveMQ Web Console or RabbitMQ Management UI)
- Network access to broker endpoints for connectivity testing

---

## Usage Examples

```
# List all brokers
aws mq list-brokers

# Check broker details
aws mq describe-broker --broker-id b-1234-5678-9012

# Check storage usage
aws cloudwatch get-metric-statistics --namespace AWS/AmazonMQ --metric-name StorePercentUsage --dimensions Name=Broker,Value=my-broker --start-time 2024-01-01T00:00:00Z --end-time 2024-01-02T00:00:00Z --period 300 --statistics Maximum

# Check broker logs
aws logs filter-log-events --log-group-name /aws/amazonmq/broker/b-1234-5678-9012/general
```

---

## License

MIT-0
