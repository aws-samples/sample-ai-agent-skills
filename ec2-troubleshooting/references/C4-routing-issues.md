---
title: "C4 — VPC Routing Issues"
description: "Diagnose connectivity failures caused by missing or incorrect VPC route table entries"
status: active
severity: HIGH
triggers:
  - "no route to host"
  - "network unreachable"
  - "blackhole.*route"
  - "cannot reach.*internet"
owner: devops-agent
objective: "Identify the routing gap and restore network path"
context: "VPC route tables control traffic flow. Each subnet is associated with one route table. Routes determine where traffic is sent: IGW (internet), NAT gateway, VPN, peering, TGW, or local VPC. Missing or blackhole routes cause connectivity failures."
---

## Phase 1 — Triage

MUST:
- Get the subnet's route table: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check for required routes based on the traffic destination:
  - Internet: 0.0.0.0/0 → IGW (public subnet) or NAT gateway (private subnet)
  - Peered VPC: peer CIDR → pcx-xxx
  - On-premises: on-prem CIDR → vgw-xxx or tgw-xxx
  - Other VPC via TGW: VPC CIDR → tgw-xxx
- Check for blackhole routes (target resource deleted but route remains)
- Verify the local route (VPC CIDR → local) exists

SHOULD:
- Check if NAT gateway is in a public subnet with IGW route
- Verify IGW is attached to the VPC
- Check Transit Gateway route tables if using TGW

MAY:
- Use VPC Reachability Analyzer for end-to-end path validation
- Check VPC flow logs for traffic patterns

## Common Issues

- symptoms: "Instance in private subnet cannot reach internet"
  diagnosis: "Missing 0.0.0.0/0 → NAT gateway route, or NAT gateway is in a private subnet."
  resolution: "Add route 0.0.0.0/0 → nat-xxx. Ensure NAT gateway's subnet has 0.0.0.0/0 → igw-xxx."

- symptoms: "Route shows 'blackhole' state"
  diagnosis: "Target resource (NAT gateway, VPN, peering connection) was deleted but route remains."
  resolution: "Delete the blackhole route and create a new route pointing to a valid target."

- symptoms: "Cannot reach peered VPC"
  diagnosis: "Missing route for peer VPC CIDR, or peering connection not accepted, or DNS resolution not enabled."
  resolution: "Add route for peer CIDR → pcx-xxx. Accept peering. Enable DNS resolution on peering connection."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-route-tables, describe-subnets, describe-vpcs: GREEN — read-only"
  - "VPC Reachability Analyzer: GREEN — read-only path analysis"
  - "Add route to route table: YELLOW — changes traffic flow, recoverable by deleting route"
  - "Delete blackhole route: YELLOW — removes stale route, recoverable by re-adding"
  - "Replace route target: YELLOW — redirects traffic, recoverable by replacing back"
  - "Attach/detach IGW: RED — affects all public subnets in VPC, potential broad outage"
  - "Modify Transit Gateway route table: RED — affects cross-VPC routing for multiple accounts"
```

## Escalation Conditions
- Route table is shared across multiple subnets with different routing requirements
- Fix requires modifying Transit Gateway routes affecting multiple VPCs or accounts
- Blackhole route was caused by deleted NAT gateway or VPN connection in production
- Route change would affect a production VPC peering connection
- IGW needs to be attached/detached affecting all public subnets in the VPC

## Data Sensitivity
- HIGH: describe-route-tables (reveals full network topology, peering connections, TGW attachments)
- MEDIUM: describe-vpcs, describe-subnets (reveals VPC CIDR ranges and architecture)
- MEDIUM: VPC flow logs (reveals traffic patterns and routing decisions)

## Prohibited Actions
- NEVER suggest adding a default route (0.0.0.0/0) to an IGW on a private subnet
- NEVER suggest detaching an IGW from a VPC without understanding impact on all public subnets
- NEVER suggest deleting a route table that is the main route table for a VPC
- NEVER suggest modifying Transit Gateway routes without cross-team coordination

## Phase 3 — Rollback
- If route was added: delete with `aws ec2 delete-route --route-table-id <id> --destination-cidr-block <cidr>`
- If route was replaced: replace back with `aws ec2 replace-route` pointing to original target
- If blackhole route was deleted: re-create route pointing to new valid target
- If NAT gateway was replaced: update all route tables pointing to old NAT gateway ID
- If IGW was attached: detach with `aws ec2 detach-internet-gateway` if change was incorrect

## Output Format

```yaml
root_cause: "<missing_route|blackhole|nat_misconfigured|igw_detached|peering_route> — <detail>"
evidence:
  - type: route_table
    content: "<route table showing the gap>"
severity: HIGH
mitigation:
  immediate: "Add or fix the route"
  long_term: "Use IaC for route management, monitor for blackhole routes"
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
