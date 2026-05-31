---
name: private5g-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Private 5G
  problems by analyzing network creation, radio unit activation,
  device connectivity, SIM provisioning, throughput, coverage,
  access point configuration, network health, VPC integration,
  and following structured runbooks.
  Activate when: network creation failures, radio unit issues,
  device connectivity problems, SIM provisioning errors, throughput
  degradation, coverage gaps, access point misconfiguration,
  network health alerts, VPC integration failures,
  or the user says something is wrong with Private 5G.
compatibility: >
  Requires AWS CLI or SDK access with private-networks,
  ec2, cloudwatch, cloudtrail, and iam permissions.
---

# AWS Private 5G Diagnostics

## When to use

Any AWS Private 5G investigation — network creation, network site management, radio unit activation, device connectivity, SIM provisioning, subscriber management, throughput issues, coverage problems, access point configuration, network health, or VPC integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws private-networks list-networks --query 'networks[*].{Arn:networkArn,Name:networkName,Status:status}'
aws private-networks list-network-sites --network-arn <arn> --query 'networkSites[*].{Arn:networkSiteArn,Name:networkSiteName,Status:status}'
aws private-networks list-network-resources --network-arn <arn> --query 'networkResources[*].{Arn:networkResourceArn,Type:type,Status:status}'
```

### Step 2 — Domain deep dive

```
aws private-networks get-network --network-arn <arn>
aws private-networks get-network-site --network-site-arn <arn>
aws private-networks get-network-resource --network-resource-arn <arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=private-networks.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Private5G --metric-name <metric> --dimensions Name=NetworkArn,Value=<arn> --start-time <start> --end-time <end> --period 300 --statistics Average
aws private-networks list-device-identifiers --network-arn <arn>
```

Read `references/guardrails.md` before concluding on any Private 5G issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `private-networks list-networks` | List all private networks |
| `private-networks get-network` | Get network details |
| `private-networks list-network-sites` | List network sites |
| `private-networks get-network-site` | Get site details |
| `private-networks list-network-resources` | List radio units and resources |
| `private-networks get-network-resource` | Get resource details |
| `private-networks list-device-identifiers` | List SIM/device identifiers |
| `private-networks list-orders` | List hardware orders |

## Gotchas: AWS Private 5G

- Private 5G is a managed CBRS (Citizens Broadband Radio Service) network. It operates on CBRS Band 48 (3.5 GHz) spectrum in the US only.
- Radio units must be physically installed and powered on before they can be activated in the console. Activation is not purely software-based.
- SIM cards are shipped by AWS and must be physically inserted into devices. You cannot use third-party SIMs.
- Network creation requires a valid shipping address for hardware delivery. The address must be in a supported AWS Private 5G region.
- Device connectivity depends on both the radio unit being active AND the SIM being provisioned. Both must be in ACTIVE status.
- Throughput is affected by physical factors: distance from radio unit, obstacles, interference, and number of connected devices.
- VPC integration uses a network interface in your VPC subnet. The subnet must have available IP addresses and proper security group rules.

## Anti-hallucination rules

1. Always cite specific network ARNs, resource ARNs, or API responses as evidence.
2. Private 5G is US-only (CBRS spectrum). Never suggest it works in other countries.
3. Hardware (radio units, SIMs) is physical. Never suggest software-only activation.
4. SIMs are AWS-provided. Never suggest using third-party SIM cards.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Network | A1–A2 | Network creation failures, network site issues |
| B — Radio & Devices | B1–B2 | Radio unit activation, device connectivity |
| C — SIM & Subscribers | C1–C2 | SIM provisioning, subscriber management |
| D — Performance | D1–D2 | Throughput issues, coverage problems |
| E — Configuration | E1–E2 | Access point configuration, network health |
| F — Integration | F1 | Integration with VPC |
| Z — Catch-All | Z1 | General troubleshooting |
