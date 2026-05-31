---
title: "A2 — Blackhole Routes"
description: "Diagnose connectivity failures caused by blackhole route entries"
status: active
severity: HIGH
triggers:
  - "blackhole"
  - "route target deleted"
  - "Connection timed out after route change"
owner: devops-agent
objective: "Identify and fix blackhole routes that silently drop traffic"
context: "A blackhole route occurs when the route target (NAT gateway, peering connection, TGW attachment, VPN) is deleted but the route entry remains. Traffic matching the route is silently dropped."
---

## Phase 1 — Triage

MUST:
- List all route tables in the VPC: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`
- Look for routes with State=blackhole in the output
- Identify which target was deleted (NatGatewayId, VpcPeeringConnectionId, TransitGatewayId)

SHOULD:
- Check CloudTrail for recent deletion of the route target
- Verify if the target was intentionally deleted or accidentally
- Check if other route tables reference the same deleted target

MAY:
- Check if an auto-scaling event or IaC deployment deleted the target
- Verify TGW route table propagation status

## Common Issues

- symptoms: "Internet access stopped working, route shows blackhole"
  diagnosis: "NAT gateway was deleted but 0.0.0.0/0 route still points to it."
  resolution: "Create a new NAT gateway and update the route, or delete the blackhole route."

- symptoms: "Cross-VPC traffic stopped, peering route is blackhole"
  diagnosis: "VPC peering connection was deleted or rejected."
  resolution: "Re-establish peering connection and update route, or remove the blackhole route."

## Safety Ratings

```
safety_ratings:
  - "describe-route-tables: GREEN — read-only inspection of route entries and blackhole state"
  - "describe-nat-gateways: GREEN — read-only check of NAT gateway state"
  - "describe-vpc-peering-connections: GREEN — read-only peering status check"
  - "delete-route (remove blackhole): YELLOW — removes a non-functional route, recoverable by re-adding"
  - "create-route (replace blackhole): YELLOW — adds route with new target, recoverable by deleting"
  - "create-nat-gateway: YELLOW — creates new NAT gateway to replace deleted target, recoverable by deleting"
```

## Escalation Conditions

- "Fix requires modifying production VPC route tables"
- "Multiple route tables reference the same deleted target"
- "Blackhole route affects default route (0.0.0.0/0) — all internet traffic impacted"
- "Fix requires creating new NAT gateway, peering connection, or TGW attachment"

## Data Sensitivity

- HIGH: route table entries (expose network architecture and traffic paths)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: NAT gateway and peering connection configurations

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete a blackhole route without confirming a replacement target is ready"

## Phase 3 — Rollback

- If a blackhole route was deleted via `delete-route`: re-add it with `aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block <cidr> --gateway-id <target>` (only if a valid target is restored)
- If a new route was created to replace the blackhole: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
- If a new NAT gateway was created: delete it with `aws ec2 delete-nat-gateway --nat-gateway-id <id>` and release the EIP
- Document the original route table state and deleted target IDs before making changes

## Output Format

```yaml
root_cause: "blackhole_route — <deleted target type and ID>"
evidence:
  - type: route_table
    content: "<route table ID, CIDR, and blackhole state>"
severity: HIGH
mitigation:
  immediate: "Replace blackhole route with valid target or delete it"
  long_term: "Monitor route table state, use IaC with dependency management"
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
