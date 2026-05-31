---
title: "D2 — VPC Configuration"
description: "Diagnose VPC-level issues affecting WorkSpaces Web sessions"
status: active
severity: HIGH
triggers: ["VPC error", "subnet issue", "routing problem", "NAT gateway", "DNS resolution failure"]
owner: devops-agent
objective: "Resolve VPC configuration issues that prevent browser sessions from accessing resources"
context: "Browser sessions run within the VPC defined in network settings. VPC routing, DNS, NAT, security groups, and NACLs all affect session connectivity."
---

## Phase 1 — Triage

MUST:
- Check VPC configuration: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[*].{VpcId:VpcId,CidrBlock:CidrBlock,State:State}'`
- Check route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id> --query 'RouteTables[*].Routes'`
- Check NAT gateway status (if using private subnets): `aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc-id> --query 'NatGateways[*].{NatGatewayId:NatGatewayId,State:State,SubnetId:SubnetId}'`

SHOULD:
- Verify DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport` and `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Check NACLs for the session subnets: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify internet gateway attachment: `aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=<vpc-id>`

MAY:
- Enable VPC flow logs for detailed traffic analysis
- Check for VPC peering connections or Transit Gateway attachments
- Verify VPC endpoint configurations

## Phase 2 — Remediate

MUST:
- Ensure route tables have appropriate routes (0.0.0.0/0 via NAT GW for private subnets)
- Verify NAT gateway is in AVAILABLE state and in a public subnet
- Enable DNS support and DNS hostnames on the VPC

SHOULD:
- Ensure NACLs allow outbound traffic on required ports (443, 80)
- Verify the NAT gateway's Elastic IP is not exhausted or throttled
- Check that the internet gateway is attached and route table has IGW route for public subnets

MAY:
- Create VPC endpoints for frequently accessed AWS services
- Configure custom DNS (Route 53 Resolver) for internal domain resolution
- Set up VPC peering for cross-VPC resource access

## Safety Ratings

```
safety_ratings:
  - "describe-vpcs: GREEN — read-only VPC details"
  - "describe-route-tables: GREEN — read-only routing information"
  - "describe-nat-gateways: GREEN — read-only NAT gateway status"
  - "create-route: RED — modifies VPC routing, affects all resources in subnet"
  - "create-nat-gateway: YELLOW — creates billable resource"
```

## Escalation Conditions

- NAT gateway is in FAILED state and cannot be recreated
- VPC CIDR conflicts with target network ranges
- Route table changes affect other workloads in the same VPC
- Cross-account VPC peering or Transit Gateway changes required

## Data Sensitivity

- `aws ec2 describe-vpcs`: LOW — returns VPC metadata
- `aws ec2 describe-route-tables`: MEDIUM — reveals network routing architecture
- `aws ec2 describe-nat-gateways`: LOW — returns NAT gateway metadata
- VPC flow logs: HIGH — contain source/destination IPs and traffic patterns

## Prohibited Actions

- NEVER delete or modify route tables without understanding the impact on all resources
- NEVER remove the NAT gateway if other resources depend on it
- NEVER modify VPC CIDR blocks on a VPC with active resources
- NEVER disable DNS support on a VPC with active WorkSpaces Web sessions

## Phase 3 — Rollback

1. Revert route table changes: delete added routes or restore removed routes
2. If a NAT gateway was created in error, delete it (after removing route table references)
3. Revert NACL changes if they affected other resources
4. Restore DNS settings if they were modified
5. Verify all resources in the VPC are functioning correctly after rollback

## Common Issues

- symptoms: "Sessions cannot access any external websites"
  diagnosis: "Private subnets have no route to NAT gateway or NAT gateway is not in AVAILABLE state."
  resolution: "Create or fix NAT gateway in a public subnet. Add 0.0.0.0/0 route via NAT gateway to private subnet route table."

## Output Format

```yaml
root_cause: "vpc_configuration — <specific_cause>"
evidence:
  - type: vpc_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC routing or DNS configuration"
  long_term: "Document VPC architecture and implement network monitoring"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
