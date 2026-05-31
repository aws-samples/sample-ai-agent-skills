# AWS IoT Core Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT Core problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT Core when the console alone isn't enough — MQTT connectivity, certificate management, IoT policies, rule engine configuration, device shadows, thing registry, custom authorizers, message throughput, throttling, and service integrations with Lambda, S3, DynamoDB, and Kinesis.

### Activate When

- MQTT connection failures or timeouts
- Certificate authentication errors
- IoT policy denying operations
- Rule engine not triggering or errors
- Rule action failures (Lambda, S3, DynamoDB, Kinesis)
- SQL query in rules not matching messages
- Device shadow not syncing
- Shadow version conflicts
- Thing registration failures
- Thing group membership issues
- Certificate lifecycle management problems
- IoT policy misconfiguration
- Custom authorizer failures
- Message throughput limitations
- Throttling on publish/subscribe
- Lambda action integration errors
- S3/DynamoDB/Kinesis action failures
- General IoT Core errors without clear symptoms

---

## Skill Structure

```
iotcore-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-mqtt-connection-failures.md
    ├── A2-certificate-issues.md
    ├── A3-policy-denied.md
    ├── B1-rule-engine-errors.md
    ├── B2-action-failures.md
    ├── B3-sql-query-issues.md
    ├── C1-shadow-sync.md
    ├── C2-shadow-conflicts.md
    ├── D1-thing-registration.md
    ├── D2-thing-groups.md
    ├── E1-certificate-management.md
    ├── E2-iot-policies.md
    ├── E3-authorizers.md
    ├── F1-message-throughput.md
    ├── F2-throttling.md
    ├── G1-lambda-actions.md
    ├── G2-s3-dynamodb-kinesis-actions.md
    ├── Z1-general-troubleshooting.md
    ├── iotcore-guardrails.md
    └── iotcore-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Connectivity** | A1–A3 | MQTT connection failures, certificate issues, policy denied |
| **B — Rules** | B1–B3 | Rule engine errors, action failures, SQL query issues |
| **C — Shadow** | C1–C2 | Device shadow sync, shadow conflicts |
| **D — Registry** | D1–D2 | Thing registration, thing groups |
| **E — Security** | E1–E3 | Certificate management, IoT policies, authorizers |
| **F — Performance** | F1–F2 | Message throughput, throttling |
| **G — Integration** | G1–G2 | Lambda actions, S3/DynamoDB/Kinesis actions |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/iotcore-guardrails.md` covering IoT policy vs IAM policy distinction, certificate lifecycle states, MQTT topic filter wildcards, device shadow versioning, rule engine SQL dialect, message broker throughput limits, custom authorizer requirements, thing group policy inheritance, reserved MQTT topics, QoS level behavior, persistent sessions, and fleet indexing requirements.

---

## Investigation Workflow

1. **Triage** — Check endpoint, thing configuration, certificate status, IoT policies
2. **Deep Dive** — Examine rule engine, device shadows, thing groups, authorizers
3. **Detailed** — Review IoT logs, CloudWatch metrics, CloudTrail events, action targets

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `iot:Describe*`, `iot:List*`, `iot:Get*`, `logs:FilterLogEvents`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- IoT logging enabled (AWSIotLogsV2 log group)
- Access to device certificates and connection details

---

## Usage Examples

```
# Get IoT data endpoint
aws iot describe-endpoint --endpoint-type iot:Data-ATS

# Check thing and its certificates
aws iot describe-thing --thing-name my-device
aws iot list-thing-principals --thing-name my-device

# Check certificate status
aws iot describe-certificate --certificate-id abc123

# Check IoT policy
aws iot get-policy --policy-name my-device-policy

# Check rule engine
aws iot get-topic-rule --rule-name my-rule

# Get device shadow
aws iot-data get-thing-shadow --thing-name my-device /dev/stdout
```

---

## License

MIT-0
