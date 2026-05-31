---
title: "G4 — VPC Peering / Transit Gateway Connectivity"
description: "Diagnose cross-VPC connectivity failures via peering or Transit Gateway"
status: active
severity: HIGH
triggers:
  - "peering.*unreachable"
  - "transit gateway.*timeout"
  - "cross-VPC.*connectivity"
owner: devops-agent
objective: "Restore cross-VPC connectivity"
context: "VPC peering is point-to-point, non-transitive. Transit Gateway is a hub for connecting multiple VPCs and on-premises networks. Both require route table entries in each VPC. Peering does not support overlapping CIDRs."
---

## Phase 1 — Triage

MUST:
- For peering: check peering connection state (must be 'active'): `aws ec2 describe-vpc-peering-connections`
- Check route tables in BOTH VPCs for routes to the peer CIDR
- Verify security groups allow traffic from the peer VPC CIDR
- Check NACLs in both subnets

SHOULD:
- For TGW: check TGW route tables: `aws ec2 describe-transit-gateway-route-tables`
- Verify TGW attachments are in 'available' state
- Check for overlapping CIDRs (peering does not support this)

MAY:
- Use VPC Reachability Analyzer for end-to-end path validation
- Check DNS resolution across peering (requires DNS resolution option enabled)

## Common Issues

- symptoms: "Peering connection active but traffic doesn't flow"
  diagnosis: "Missing routes in one or both VPC route tables."
  resolution: "Add route for peer VPC CIDR → pcx-xxx in both VPC route tables."

- symptoms: "TGW attachment available but no connectivity"
  diagnosis: "TGW route table missing route, or TGW route table not associated with the attachment."
  resolution: "Add route in TGW route table. Associate route table with the attachment."

- symptoms: "Peering fails with overlapping CIDR error"
  diagnosis: "Both VPCs have overlapping IP ranges. Peering does not support this."
  resolution: "Use different CIDRs, or use PrivateLink for specific service connectivity."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-vpc-peering-connections, describe-transit-gateway-route-tables: GREEN — read-only"
  - "VPC Reachability Analyzer: GREEN — read-only path analysis"
  - "Add route for peer VPC CIDR: YELLOW — changes routing, recoverable by deleting route"
  - "Accept peering connection: YELLOW — enables cross-VPC traffic, recoverable by deleting peering"
  - "Enable DNS resolution on peering: YELLOW — changes DNS behavior, recoverable"
  - "Modify TGW route table: RED — affects routing for multiple VPCs and accounts"
  - "Delete peering connection: RED — permanently removes cross-VPC connectivity"
```

## Escalation Conditions
- Transit Gateway route changes affect multiple VPCs or AWS accounts
- Peering connection involves VPCs in different AWS accounts requiring cross-account coordination
- Overlapping CIDRs prevent peering and require network redesign
- TGW attachment is in a shared services VPC affecting many downstream consumers
- Cross-VPC connectivity change has compliance or security audit implications

## Data Sensitivity
- HIGH: describe-vpc-peering-connections (reveals cross-VPC relationships, account IDs, CIDRs)
- HIGH: describe-transit-gateway-route-tables (reveals full network topology across VPCs)
- MEDIUM: describe-route-tables (reveals VPC routing architecture)
- MEDIUM: VPC flow logs (reveals cross-VPC traffic patterns)

## Prohibited Actions
- NEVER suggest deleting a peering connection without confirming both sides are aware
- NEVER suggest modifying TGW route tables without cross-team coordination
- NEVER suggest enabling transitive routing through peering (not supported)
- NEVER suggest overlapping CIDR workarounds that bypass VPC peering limitations

## Phase 3 — Rollback
- If route was added for peering: delete route from both VPC route tables
- If peering connection was accepted: delete peering connection from either side
- If DNS resolution was enabled on peering: disable with `modify-vpc-peering-connection-options`
- If TGW route was added: delete TGW route and disassociate route table if needed
- If TGW attachment was created: delete attachment (may take several minutes)

## Output Format

```yaml
root_cause: "<missing_route|peering_not_active|tgw_route_table|overlapping_cidr|sg_blocking>"
evidence:
  - type: peering_state
    content: "<peering or TGW state and route tables>"
severity: HIGH
mitigation:
  immediate: "Add routes and fix security groups"
  long_term: "Use IaC for cross-VPC routing, implement monitoring"
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
