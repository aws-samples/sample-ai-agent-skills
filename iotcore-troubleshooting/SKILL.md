---
name: iotcore-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT Core problems
  by analyzing MQTT connectivity, certificate management, IoT policies,
  rule engine, device shadows, thing registry, security configuration,
  message throughput, and service integrations using structured runbooks.
  Activate when: MQTT connection failures, certificate issues, policy
  denied errors, rule engine errors, action failures, SQL query issues,
  device shadow sync problems, shadow conflicts, thing registration
  errors, thing group issues, certificate management problems, IoT
  policy misconfiguration, custom authorizer failures, message throughput
  issues, throttling, Lambda action errors, S3/DynamoDB/Kinesis action
  failures, or the user says something is wrong with IoT Core without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with IoT, IAM, CloudWatch, CloudTrail,
  Lambda, S3, DynamoDB, and Kinesis permissions as applicable.
---

# AWS IoT Core Diagnostics

## When to use

Any AWS IoT Core investigation where the console alone is insufficient — MQTT connectivity, certificate and policy issues, rule engine failures, device shadow problems, thing registry management, security configuration, throughput optimization, or service integration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iot describe-endpoint --endpoint-type iot:Data-ATS
aws iot list-things --max-results 25
aws iot describe-thing --thing-name <thing-name>
aws iot list-thing-principals --thing-name <thing-name>
aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name Connect.Success --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws iot describe-certificate --certificate-id <cert-id>
aws iot list-attached-policies --target <cert-arn>
aws iot get-policy --policy-name <policy-name>
aws iot list-topic-rules --max-results 25
aws iot get-topic-rule --rule-name <rule-name>
aws iot get-thing-shadow --thing-name <thing-name>
```

### Step 3 — Detailed investigation

```
aws iot describe-authorizer --authorizer-name <name>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iot.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name AWSIotLogsV2 --start-time <epoch>
aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name RuleMessageThrottled --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/iotcore-guardrails.md` before concluding on any IoT Core issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-endpoint` | Get IoT data endpoint for connections |
| `describe-thing` | Check thing attributes and configuration |
| `list-thing-principals` | Check certificates attached to a thing |
| `describe-certificate` | Check certificate status and details |
| `get-policy` | Check IoT policy document |
| `get-topic-rule` | Check rule engine configuration |
| `get-thing-shadow` | Check device shadow state |
| IoT Logs (CloudWatch) | Check connection and message logs |
| CloudWatch Metrics | Check throughput, errors, throttling |

## Gotchas: AWS IoT Core

- IoT policies are NOT IAM policies. IoT policies are attached to certificates or Cognito identities, not IAM users/roles. They use a different policy syntax with IoT-specific actions and resource ARNs. The policy must allow the specific MQTT topics the device uses.
- Certificate must be ACTIVE and attached to a thing AND a policy. A certificate in INACTIVE state will reject connections. The certificate must be attached to both a thing (for identity) and a policy (for authorization). Missing either attachment causes connection or authorization failures.
- MQTT topic filters use / as separator and + and # as wildcards. + matches a single level, # matches all remaining levels. Topic filters in policies must match the topics devices publish/subscribe to. Case-sensitive matching. Leading $ is reserved for system topics ($aws/things/...).
- Device shadows have desired and reported states. Conflicts occur when both are updated simultaneously. Shadow version is incremented on each update. Optimistic locking uses version for conflict resolution. Classic shadows use unnamed shadow; named shadows allow multiple per thing.
- Rule engine SQL uses a specific dialect. It supports SELECT, FROM, WHERE with IoT-specific functions. The FROM clause specifies the MQTT topic filter. Errors in SQL syntax cause silent rule failures. Test rules with the IoT console test feature.
- Message broker has per-account throughput limits. Publish and subscribe operations have separate limits. Throttling returns 503 errors. Limits vary by region. Request limit increases through Service Quotas.
- Custom authorizers use Lambda for token validation. They must return an IoT policy document. Timeout is 5 seconds. Caching can reduce Lambda invocations. Signing is optional but recommended.
- Thing groups support dynamic membership. Dynamic groups use fleet indexing queries. Static groups have manual membership. Group policies apply to all things in the group. Nested groups inherit parent policies.

## Anti-hallucination rules

1. Always cite specific thing names, certificate IDs, or API responses as evidence.
2. IoT policies are NOT IAM policies. Never confuse the two policy systems.
3. Certificates must be ACTIVE. Never assume certificate status.
4. MQTT topic wildcards: + is single-level, # is multi-level. Never confuse them.
5. Device shadow version must be checked for conflicts. Never ignore version mismatches.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Connectivity | A1-A3 | MQTT connection failures, certificate issues, policy denied |
| B — Rules | B1-B3 | Rule engine errors, action failures, SQL query issues |
| C — Shadow | C1-C2 | Device shadow sync, shadow conflicts |
| D — Registry | D1-D2 | Thing registration, thing groups |
| E — Security | E1-E3 | Certificate management, IoT policies, authorizers |
| F — Performance | F1-F2 | Message throughput, throttling |
| G — Integration | G1-G2 | Lambda actions, S3/DynamoDB/Kinesis actions |
| Z — Catch-All | Z1 | General troubleshooting |
