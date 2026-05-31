---
title: "G3 — NAT Gateway / NAT Instance Issues"
description: "Diagnose outbound internet connectivity failures through NAT"
status: active
severity: HIGH
triggers:
  - "NAT.*timeout"
  - "cannot reach.*internet.*private"
  - "NAT gateway.*error"
owner: devops-agent
objective: "Restore outbound internet connectivity for private subnet instances"
context: "NAT gateways provide outbound internet access for private subnet instances. NAT gateway must be in a public subnet with an EIP. Private subnet route table must have 0.0.0.0/0 → NAT gateway. NAT gateways are AZ-specific."
---

## Phase 1 — Triage

MUST:
- Check private subnet route table: 0.0.0.0/0 → nat-xxx
- Check NAT gateway state: `aws ec2 describe-nat-gateways --nat-gateway-ids <id>` — must be 'available'
- Check NAT gateway's subnet route table: must have 0.0.0.0/0 → igw-xxx
- Verify NAT gateway has an EIP associated

SHOULD:
- Check NAT gateway CloudWatch metrics: PacketsDropCount, ErrorPortAllocation
- Verify NAT gateway is in the same AZ as the private subnet (for optimal routing)
- Check if NAT gateway port allocation is exhausted (64K ports per destination)

MAY:
- Check if NAT instance (not gateway) is being used — verify source/dest check is disabled
- Check NAT gateway bandwidth (scales to 100 Gbps but may have burst limits)

## Common Issues

- symptoms: "Private instances cannot reach internet, NAT gateway exists"
  diagnosis: "Route table missing 0.0.0.0/0 → NAT gateway, or NAT gateway's subnet lacks IGW route."
  resolution: "Add route in private subnet RT. Verify NAT gateway subnet has IGW route."

- symptoms: "NAT gateway ErrorPortAllocation > 0"
  diagnosis: "Port exhaustion. Too many connections to the same destination from the same source."
  resolution: "Use multiple NAT gateways, or allocate multiple EIPs to the NAT gateway."

- symptoms: "NAT gateway state is 'failed'"
  diagnosis: "NAT gateway creation failed. Common causes: no EIP, subnet has no IGW route, or subnet has no available IPs."
  resolution: "Delete failed NAT gateway. Fix prerequisites and create a new one."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-nat-gateways, describe-route-tables: GREEN — read-only"
  - "CloudWatch NAT gateway metrics: GREEN — read-only"
  - "Add route to NAT gateway in private subnet RT: YELLOW — changes routing, recoverable"
  - "Add additional EIP to NAT gateway: YELLOW — increases port capacity, recoverable"
  - "Create new NAT gateway: YELLOW — new resource, no impact until routes point to it"
  - "Delete failed NAT gateway: YELLOW — removes failed resource, no traffic impact"
  - "Delete working NAT gateway: RED — breaks outbound internet for all private subnet instances"
```

## Escalation Conditions
- NAT gateway port exhaustion affecting production workloads
- NAT gateway in 'failed' state and private subnet instances have no internet access
- Multiple AZs need NAT gateway changes simultaneously
- NAT gateway replacement requires EIP change affecting firewall whitelists
- NAT gateway costs are unexpectedly high and require architecture review

## Data Sensitivity
- HIGH: describe-nat-gateways (reveals EIPs, subnet placement, VPC topology)
- MEDIUM: CloudWatch NAT metrics (reveals traffic patterns, connection counts)
- MEDIUM: describe-route-tables (reveals network routing architecture)

## Prohibited Actions
- NEVER suggest deleting a working NAT gateway without first creating a replacement
- NEVER suggest using a NAT instance instead of NAT gateway without explaining limitations
- NEVER suggest placing a NAT gateway in a private subnet (requires public subnet with IGW route)
- NEVER suggest sharing a single NAT gateway across AZs for production workloads (single AZ failure risk)

## Phase 3 — Rollback
- If route was added to NAT gateway: delete route with `delete-route`
- If NAT gateway was replaced: update routes to point back to old NAT gateway (if still available)
- If additional EIP was added: remove with `disassociate-address` if port exhaustion is resolved
- If NAT gateway was deleted: create new NAT gateway and update all affected route tables

## Output Format

```yaml
root_cause: "<missing_route|nat_failed|port_exhaustion|no_igw_route> — <detail>"
evidence:
  - type: nat_gateway_state
    content: "<describe-nat-gateways output>"
severity: HIGH
mitigation:
  immediate: "Fix routing or replace NAT gateway"
  long_term: "Multi-AZ NAT gateways, monitor port allocation, use VPC endpoints for AWS services"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
