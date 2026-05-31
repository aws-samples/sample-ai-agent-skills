---
title: "D3 — NAT Gateway Port Exhaustion"
description: "Diagnose NAT gateway connectivity issues caused by port allocation exhaustion"
status: active
severity: HIGH
triggers:
  - "ErrorPortAllocation"
  - "Intermittent connection failures through NAT"
  - "NAT gateway dropping packets"
owner: devops-agent
objective: "Resolve NAT gateway port exhaustion and restore reliable connectivity"
context: "A NAT gateway supports up to 55,000 simultaneous connections per destination IP+port. If a single destination receives too many connections, ports are exhausted. ErrorPortAllocation metric indicates this."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch metric: `ErrorPortAllocation` for the NAT gateway
- Check `ActiveConnectionCount` and `PacketsDropCount` metrics
- Identify if traffic is concentrated to a single destination (e.g., one API endpoint)

SHOULD:
- Check if applications are properly closing connections
- Verify connection pooling is configured in applications
- Check if multiple NAT gateways could distribute the load

MAY:
- Analyze flow logs to identify the high-connection-count destinations
- Check if DNS TTL is causing all traffic to go to one IP of a multi-IP service

## Common Issues

- symptoms: "ErrorPortAllocation spikes, connections to S3 failing"
  diagnosis: "All S3 traffic going through NAT instead of using S3 gateway endpoint."
  resolution: "Add S3 gateway endpoint (free) to avoid NAT for S3 traffic."

- symptoms: "Thousands of connections to single API endpoint"
  diagnosis: "55,000 port limit per destination IP+port exceeded."
  resolution: "Use multiple NAT gateways, implement connection pooling, or use VPC endpoints."

## Safety Ratings

```
safety_ratings:
  - "get-metric-statistics (ErrorPortAllocation): GREEN — read-only CloudWatch metric query"
  - "get-metric-statistics (ActiveConnectionCount): GREEN — read-only metric query"
  - "describe-nat-gateways: GREEN — read-only NAT gateway inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "create-nat-gateway (add additional NAT): YELLOW — creates new NAT gateway, recoverable by deleting"
  - "create-route (distribute traffic to new NAT): YELLOW — adds route, recoverable by deleting"
  - "create-vpc-endpoint (add S3/DynamoDB gateway endpoint): YELLOW — creates endpoint, recoverable by deleting"
```

## Escalation Conditions

- "Fix requires creating additional NAT gateways in production VPC"
- "Fix requires modifying production VPC route tables"
- "Port exhaustion affecting multiple services and applications"
- "Fix requires application-level changes (connection pooling)"

## Data Sensitivity

- HIGH: VPC flow logs (contain source/destination IPs and ports)
- HIGH: NAT gateway traffic patterns (expose which external services are accessed)
- MEDIUM: CloudWatch metrics, NAT gateway configuration

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete an existing NAT gateway to resolve port exhaustion"

## Phase 3 — Rollback

- If a new NAT gateway was created: delete it with `aws ec2 delete-nat-gateway --nat-gateway-id <id>` and release the EIP
- If routes were changed to distribute traffic: replace them back with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <original-nat-id>`
- If a VPC endpoint was created: delete it with `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <vpce-id>`
- Document original NAT gateway configuration and route tables before changes

## Output Format

```yaml
root_cause: "nat_port_exhaustion — <detail>"
evidence:
  - type: cloudwatch_metric
    content: "<ErrorPortAllocation count and ActiveConnectionCount>"
severity: HIGH
mitigation:
  immediate: "Add additional NAT gateways or use VPC endpoints for AWS services"
  long_term: "Implement connection pooling, use VPC endpoints, distribute across NAT gateways"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
