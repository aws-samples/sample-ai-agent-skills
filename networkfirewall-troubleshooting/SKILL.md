---
name: networkfirewall-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Network Firewall problems
  by analyzing firewall deployment, rule configuration, routing, logging,
  performance, TLS inspection, and integration issues following structured
  runbooks. Activate when: firewall creation failures, deployment issues,
  endpoint problems, stateless rule mismatches, stateful rule errors, rule
  order conflicts, route table misconfiguration, asymmetric routing, inspection
  VPC design issues, alert log analysis, flow log problems, throughput
  degradation, latency issues, TLS inspection failures, certificate management
  errors, Firewall Manager integration problems, Transit Gateway integration
  issues, or the user says something is wrong with Network Firewall without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with NetworkFirewall, EC2, IAM, CloudWatch,
  CloudTrail, S3, and optionally Firewall Manager and Transit Gateway
  permissions.
---

# AWS Network Firewall Diagnostics

## When to use

Any AWS Network Firewall investigation where the console alone is insufficient — firewall deployment, rule configuration (stateless and stateful), routing and traffic inspection, logging and alerting, performance optimization, TLS inspection, or integration with Firewall Manager and Transit Gateway.

## Investigation workflow

### Step 1 — Collect and triage

```
aws network-firewall list-firewalls --query 'Firewalls[*].{Name:FirewallName,Arn:FirewallArn}'
version: "1.0.0"
last_updated: "2025-04-12"
aws network-firewall describe-firewall --firewall-name <firewall-name>
aws network-firewall describe-firewall-policy --firewall-policy-name <policy-name>
aws network-firewall describe-logging-configuration --firewall-arn <firewall-arn>
```

### Step 2 — Domain deep dive

```
aws network-firewall describe-rule-group --rule-group-name <rule-group-name> --type STATEFUL
aws network-firewall describe-rule-group --rule-group-name <rule-group-name> --type STATELESS
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>
aws ec2 describe-vpc-endpoints --filters Name=vpc-endpoint-type,Values=GatewayLoadBalancer
aws network-firewall list-rule-groups --type STATEFUL
aws network-firewall list-rule-groups --type STATELESS
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=network-firewall.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name <firewall-alert-log-group> --filter-pattern "{ $.event.alert }"
aws cloudwatch get-metric-statistics --namespace AWS/NetworkFirewall --metric-name DroppedPackets --dimensions Name=FirewallName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/networkfirewall-guardrails.md` before concluding on any Network Firewall issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-firewall` | Check firewall status, subnets, policy |
| `describe-firewall-policy` | Check policy configuration and rule group references |
| `describe-rule-group` | Check stateless/stateful rule definitions |
| `describe-logging-configuration` | Check alert and flow log destinations |
| `describe-route-tables` | Verify traffic routing through firewall endpoints |
| CloudWatch Metrics | Monitor packets, bytes, drops, passes |
| CloudWatch Logs / S3 | Analyze alert and flow logs |
| CloudTrail | Audit configuration changes |

## Gotchas: AWS Network Firewall

- Network Firewall creates endpoints in specified subnets. These are Gateway Load Balancer endpoints (GWLBe). Traffic must be routed TO these endpoints via route tables. The firewall does NOT automatically intercept traffic — routing must be explicitly configured. This is the #1 source of "firewall not working" issues.
- Stateless rules are evaluated BEFORE stateful rules. Stateless rules process packets individually (no connection tracking). Stateful rules process flows (connection tracking). If a stateless rule drops or passes a packet, stateful rules never see it. Use stateless for simple allow/deny; stateful for protocol-aware inspection.
- Stateful rule order matters. In STRICT_ORDER mode, rules are evaluated in priority order (lowest number first). In DEFAULT_ACTION_ORDER mode, pass rules are evaluated first, then drop, then alert. The evaluation order significantly affects which rule matches. Choose the mode carefully.
- Asymmetric routing breaks stateful inspection. Stateful rules track connections. If request traffic goes through the firewall but response traffic bypasses it (or vice versa), stateful inspection fails. Ensure both directions of a flow traverse the same firewall endpoint. This is critical in multi-AZ and Transit Gateway designs.
- Firewall subnets should be dedicated. Do not place workloads in firewall subnets. Firewall subnets should contain only the firewall endpoints. Workload subnets route traffic to firewall endpoints. Mixing workloads and firewall endpoints in the same subnet causes routing loops.
- TLS inspection requires certificate management. TLS inspection decrypts, inspects, and re-encrypts HTTPS traffic. It requires a CA certificate in ACM. Clients must trust the CA certificate. TLS inspection adds latency and has throughput implications. Not all TLS versions and cipher suites are supported.
- Alert logs and flow logs serve different purposes. Alert logs contain entries when stateful rules match (alert or drop actions). Flow logs contain all traffic flow records. Both are optional and must be explicitly enabled. Logs can go to CloudWatch Logs, S3, or Kinesis Data Firehose.

## Anti-hallucination rules

1. Always cite specific firewall names, rule group names, or API responses as evidence.
2. Routing is REQUIRED to send traffic through firewall. Never claim auto-interception.
3. Stateless rules evaluate BEFORE stateful. Never reverse the order.
4. Firewall subnets should NOT contain workloads. Never suggest co-locating.
5. Asymmetric routing breaks stateful inspection. Never ignore routing symmetry.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Firewall | A1–A3 | Creation failures, deployment, endpoint issues |
| B — Rules | B1–B3 | Stateless rules, stateful rules, rule order |
| C — Routing | C1–C3 | Route table config, asymmetric routing, inspection VPC |
| D — Logging | D1–D2 | Alert logs, flow logs |
| E — Performance | E1–E2 | Throughput, latency |
| F — TLS | F1–F2 | TLS inspection, certificate management |
| G — Integration | G1–G2 | Firewall Manager, Transit Gateway |
| Z — Catch-All | Z1 | General troubleshooting |
