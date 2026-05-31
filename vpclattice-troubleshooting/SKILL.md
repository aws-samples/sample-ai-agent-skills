---
name: vpclattice-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon VPC Lattice problems
  by analyzing service networks, services, target groups, listener rules,
  auth policies, cross-account sharing, DNS resolution, health checks,
  access logs, and VPC associations following structured runbooks. Activate
  when: service network creation failures, service registration issues,
  target group health failures, listener rule mismatches, auth policy denials,
  cross-account sharing problems, DNS resolution failures, health check
  failures, missing access logs, VPC association errors, or the user says
  something is wrong with VPC Lattice without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with vpc-lattice, EC2, IAM, CloudWatch,
  CloudTrail, S3, RAM, and Route 53 permissions.
---

# Amazon VPC Lattice Diagnostics

## When to use

Any Amazon VPC Lattice investigation where the console alone is insufficient — service network management, service creation and configuration, target group health, listener rules and routing, auth policies, cross-account sharing via RAM, DNS resolution, health checks, access logging, or VPC association issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws vpc-lattice list-service-networks
aws vpc-lattice list-services
aws vpc-lattice list-target-groups
aws vpc-lattice get-service-network --service-network-identifier <id>
aws vpc-lattice get-service --service-identifier <id>
```

### Step 2 — Domain deep dive

```
aws vpc-lattice list-service-network-service-associations --service-network-identifier <id>
aws vpc-lattice list-service-network-vpc-associations --service-network-identifier <id>
aws vpc-lattice list-targets --target-group-identifier <id>
aws vpc-lattice list-listeners --service-identifier <id>
aws vpc-lattice list-rules --service-identifier <id> --listener-identifier <id>
aws vpc-lattice get-auth-policy --resource-identifier <id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=vpc-lattice.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name <lattice-access-log-group> --filter-pattern "{ $.responseCode = 4* }"
aws cloudwatch get-metric-statistics --namespace AWS/VPCLattice --metric-name RequestCount --dimensions Name=ServiceName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any VPC Lattice issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-service-network` | Check service network status and config |
| `get-service` | Check service status, DNS, custom domain |
| `list-targets` | Check target registration and health |
| `get-auth-policy` | Check auth policy on service network or service |
| `list-service-network-vpc-associations` | Check VPC associations |
| `get-access-log-subscription` | Check access log configuration |
| CloudWatch Metrics | Monitor request counts, latency, errors |
| CloudTrail | Audit configuration changes |

## Gotchas: Amazon VPC Lattice

- VPC Lattice uses a managed DNS name for each service. Clients in associated VPCs resolve the service DNS name automatically via the VPC Lattice managed prefix (169.254.171.0/24 and fd00:ec2:80::/48). If DNS resolution fails, check VPC association status and DNS settings.
- Auth policies are evaluated at TWO levels: service network AND service. Both must allow the request. A deny at either level blocks the request. Auth policies use IAM policy syntax but are NOT IAM policies — they are VPC Lattice resource policies.
- Target groups support IP, Instance, Lambda, and ALB target types. Each type has different health check behavior. Lambda targets do not support health checks. ALB targets require the ALB to be in a VPC associated with the service network.
- Cross-account sharing uses AWS RAM. The service network owner shares via RAM, and the consumer account accepts the share and associates their VPC. Services can also be shared cross-account and associated with shared service networks.
- Access logs must be explicitly enabled per service network. Logs can go to CloudWatch Logs, S3, or Kinesis Data Firehose. Without access logs, you cannot see request/response details.
- Listener rules use path-based and header-based matching. Rules are evaluated in priority order (lowest number first). The default rule catches unmatched requests. Weighted target groups enable traffic splitting.

## Anti-hallucination rules

1. Always cite specific service network IDs, service IDs, or API responses as evidence.
2. Auth policies exist at BOTH service network and service level. Never ignore either.
3. VPC Lattice DNS requires VPC association. Never claim automatic DNS resolution without association.
4. Lambda targets do NOT support health checks. Never suggest configuring them.
5. Access logs are NOT enabled by default. Never assume they exist.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Service Network | A1–A2 | Creation failures, configuration |
| B — Services | B1–B2 | Service creation, listener config |
| C — Target Groups | C1–C2 | Target registration, health issues |
| D — Listener Rules | D1–D2 | Rule matching, routing issues |
| E — Auth Policies | E1–E2 | Policy evaluation, cross-level auth |
| F — Cross-Account | F1–F2 | RAM sharing, consumer association |
| G — DNS & Health | G1–G2 | DNS resolution, health checks |
| H — Logging & VPC | H1–H2 | Access logs, VPC association |
| Z — Catch-All | Z1 | General troubleshooting |
