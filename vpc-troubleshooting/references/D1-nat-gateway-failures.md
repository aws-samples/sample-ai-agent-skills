---
title: "D1 — NAT Gateway Failures"
description: "Diagnose NAT gateway connectivity and operational failures"
status: active
severity: CRITICAL
triggers:
  - "NAT gateway failed"
  - "No internet from private subnet"
  - "NAT gateway state not available"
owner: devops-agent
objective: "Restore NAT gateway functionality and internet access for private subnets"
context: "NAT gateway must be in a PUBLIC subnet (with IGW route). Private subnets route 0.0.0.0/0 → NAT. NAT gateway needs an Elastic IP. It's AZ-specific — one per AZ for high availability."
---

## Phase 1 — Triage

MUST:
- Check NAT gateway state: `aws ec2 describe-nat-gateways --nat-gateway-ids <id>` → State (available, failed, deleted)
- Verify NAT gateway is in a PUBLIC subnet (subnet's route table has 0.0.0.0/0 → IGW)
- Verify NAT gateway has an Elastic IP associated
- Check private subnet route table has 0.0.0.0/0 → NAT gateway

SHOULD:
- Check NAT gateway CloudWatch metrics: PacketsDropCount, ErrorPortAllocation
- Verify the IGW is attached to the VPC
- Check if the NAT gateway's subnet NACL allows required traffic

MAY:
- Check if NAT gateway was recently created (takes a few minutes to become available)
- Verify there are available EIPs in the account (5 per region default)

## Common Issues

- symptoms: "NAT gateway state is 'failed'"
  diagnosis: "NAT gateway creation failed. Common causes: EIP already associated, subnet is not public, no IGW."
  resolution: "Delete failed NAT, fix the root cause, create a new NAT gateway."

- symptoms: "NAT gateway is 'available' but private instances can't reach internet"
  diagnosis: "Private subnet route table missing 0.0.0.0/0 → NAT gateway, or NACL blocking."
  resolution: "Add default route to NAT gateway in private subnet route table."

## Safety Ratings

```
safety_ratings:
  - "describe-nat-gateways: GREEN — read-only NAT gateway state inspection"
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "describe-subnets: GREEN — read-only subnet configuration check"
  - "create-nat-gateway: YELLOW — creates new NAT gateway, recoverable by deleting"
  - "create-route (add default route to NAT): YELLOW — adds route entry, recoverable by deleting"
  - "delete-nat-gateway: RED — deletes NAT gateway, breaks internet access for all private subnets using it"
```

## Escalation Conditions

- "Fix requires creating or replacing NAT gateway in production VPC"
- "Fix requires modifying production VPC route tables"
- "Multiple subnets or AZs affected by NAT gateway failure"
- "NAT gateway failure impacts all private subnet internet access"

## Data Sensitivity

- HIGH: route table entries (expose network architecture and traffic paths)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: NAT gateway configuration, Elastic IP associations, subnet CIDR ranges

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete a NAT gateway without confirming a replacement is available"

## Phase 3 — Rollback

- If a new NAT gateway was created: delete it with `aws ec2 delete-nat-gateway --nat-gateway-id <id>` and release the EIP with `aws ec2 release-address --allocation-id <eip-alloc-id>`
- If a default route was added to NAT: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0`
- If a route was changed to point to new NAT: replace it back with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <original-nat-id>`
- Document NAT gateway ID, EIP, subnet, and route table state before making changes

## Output Format

```yaml
root_cause: "nat_gateway_failure — <detail>"
evidence:
  - type: nat_gateway
    content: "<NAT gateway state and configuration>"
severity: CRITICAL
mitigation:
  immediate: "Fix NAT gateway configuration or create new NAT gateway"
  long_term: "Deploy NAT gateway per AZ, monitor with CloudWatch alarms"
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
