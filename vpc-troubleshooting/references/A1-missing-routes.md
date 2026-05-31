---
title: "A1 — Missing Routes"
description: "Diagnose connectivity failures caused by missing route table entries"
status: active
severity: HIGH
triggers:
  - "No route to host"
  - "Connection timed out"
  - "REJECT in flow logs"
  - "Reachability Analyzer: no route"
owner: devops-agent
objective: "Identify missing routes and restore connectivity"
context: "Every subnet is associated with exactly one route table. If no explicit association, the VPC main route table is used. Missing routes cause silent packet drops — no ICMP unreachable is returned."
---

## Phase 1 — Triage

MUST:
- Identify source and destination resources and their subnets
- Check route table for source subnet: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check if destination CIDR has a matching route entry
- If no explicit association, check main route table: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id> Name=association.main,Values=true`

SHOULD:
- Run Reachability Analyzer for end-to-end path validation
- Check VPC flow logs for REJECT entries on the source ENI
- Verify the route target exists and is in available state (IGW, NAT, peering, TGW)

MAY:
- Check if route was recently deleted via CloudTrail
- Verify route propagation settings for VPN/TGW routes

## Common Issues

- symptoms: "Instance can't reach internet, no 0.0.0.0/0 route"
  diagnosis: "Subnet route table missing default route to IGW or NAT gateway."
  resolution: "Add 0.0.0.0/0 → IGW (public subnet) or 0.0.0.0/0 → NAT gateway (private subnet)."

- symptoms: "Can't reach resources in peered VPC"
  diagnosis: "Route table missing entry for peer VPC CIDR pointing to peering connection."
  resolution: "Add route for peer VPC CIDR → pcx-xxx in BOTH VPCs' route tables."

- symptoms: "Route exists but target shows blackhole"
  diagnosis: "Route target was deleted (NAT gateway, peering connection, TGW) but route entry remains."
  resolution: "Delete the blackhole route and create a new route with a valid target."

## Safety Ratings

```
safety_ratings:
  - "describe-route-tables: GREEN — read-only inspection of route table entries"
  - "describe-vpn-gateways: GREEN — read-only check of VPN gateway state"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "create-route (add missing route): YELLOW — adds a new route entry, recoverable by deleting the route"
  - "replace-route: YELLOW — modifies existing route target, recoverable by replacing back to original target"
```

## Escalation Conditions

- "Fix requires modifying production VPC route tables"
- "Multiple subnets or AZs affected by missing routes"
- "Missing route involves Transit Gateway or VPN route propagation"
- "Fix requires adding a default route (0.0.0.0/0) — affects all outbound traffic"

## Data Sensitivity

- HIGH: route table entries (expose network architecture and traffic paths)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: subnet associations and VPC CIDR ranges

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER add a route without verifying the target resource exists and is available"

## Phase 3 — Rollback

- If a route was added via `create-route`: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
- If a route was replaced via `replace-route`: replace it back to the original target with `aws ec2 replace-route --route-table-id <rtb-id> --destination-cidr-block <cidr> --gateway-id <original-target>`
- Document the original route table state before making changes

## Output Format

```yaml
root_cause: "missing_route — <detail>"
evidence:
  - type: route_table
    content: "<route table ID and missing entry>"
severity: HIGH
mitigation:
  immediate: "Add the missing route entry"
  long_term: "Use infrastructure-as-code to manage route tables, enable route propagation where applicable"
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
