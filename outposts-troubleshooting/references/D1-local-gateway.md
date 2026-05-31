---
title: "D1 — Local Gateway Configuration Issues"
description: "Diagnose local gateway routing and on-premises connectivity problems"
status: active
severity: HIGH
triggers:
  - "local gateway"
  - "LGW"
  - "on-premises connectivity"
  - "CoIP"
  - "customer-owned IP"
owner: devops-agent
objective: "Resolve local gateway routing, CoIP pool, and on-premises network connectivity issues"
context: "The local gateway (LGW) provides connectivity between Outpost subnets and the on-premises network. It requires explicit configuration including route table associations, CoIP pool allocation, and optionally BGP peering."
---

## Phase 1 — Triage

MUST:
- List local gateways: `aws ec2 describe-local-gateways`
- Check route tables: `aws ec2 describe-local-gateway-route-tables`
- Check VPC associations: `aws ec2 describe-local-gateway-route-table-vpc-associations`
- Check virtual interfaces: `aws ec2 describe-local-gateway-virtual-interfaces`
- Check CoIP pools: `aws ec2 describe-coip-pools`

SHOULD:
- Check route table entries: `aws ec2 search-local-gateway-routes --local-gateway-route-table-id <lgw-rtb-id> --filters "Name=type,Values=static,propagated"`
- Verify CoIP address allocations: `aws ec2 describe-addresses --filters "Name=domain,Values=vpc" "Name=public-ip-source,Values=customer-owned-ip-pool"`
- Check virtual interface group associations: `aws ec2 describe-local-gateway-route-table-virtual-interface-group-associations`

MAY:
- Verify BGP peering status with on-premises routers
- Check on-premises firewall rules for Outpost traffic

## Phase 2 — Remediate

MUST:
- If no VPC association: create one: `aws ec2 create-local-gateway-route-table-vpc-association --local-gateway-route-table-id <lgw-rtb-id> --vpc-id <vpc-id>`
- If no routes: add static routes or verify BGP propagation
- If CoIP not allocated: allocate addresses from the CoIP pool

SHOULD:
- Verify subnet route tables include routes to the local gateway for on-premises CIDRs
- Ensure security groups allow traffic to/from on-premises networks
- Test connectivity with ping or traceroute from Outpost instances

MAY:
- Implement redundant virtual interfaces for high availability
- Document the local gateway configuration for operational reference

## Common Issues

- symptoms: "Outpost instances cannot reach on-premises servers"
  diagnosis: "Local gateway route table not associated with the VPC or missing routes."
  resolution: "Create VPC association and add routes for on-premises CIDRs."

- symptoms: "On-premises servers cannot reach Outpost instances"
  diagnosis: "CoIP addresses not allocated or on-premises routing not configured."
  resolution: "Allocate CoIP addresses and configure on-premises routes to the CoIP range."

## Output Format

```yaml
root_cause: "local_gateway — <specific_cause>"
evidence:
  - type: lgw_config
    content: "<local gateway route table and associations>"
  - type: coip_pools
    content: "<CoIP pool allocation>"
severity: HIGH
mitigation:
  immediate: "Fix local gateway routing or CoIP configuration"
  long_term: "Implement redundant virtual interfaces and monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — local gateway controls network access to on-premises resources
- Service Disruption Risk: HIGH — gateway issues affect all local network connectivity
- Compliance Impact: HIGH — local gateway security affects network isolation

## Escalation Conditions
- Local gateway becomes completely unavailable
- Routing table changes break local network connectivity
- CoIP pool exhaustion prevents new address assignments
- Local gateway route propagation fails
- Security group changes on local gateway block traffic

## Data Sensitivity
- Local gateway configurations — INTERNAL
- Routing table entries — INTERNAL
- CoIP pool allocations — INTERNAL
- VPC association details — INTERNAL
- Never expose local network topology externally

## Prohibited Actions
- Do NOT modify local gateway routing without network team coordination
- Do NOT delete local gateway route tables with active associations
- Do NOT exhaust CoIP pools without capacity planning
- Do NOT bypass local gateway security controls
- Do NOT expose local gateway management to untrusted networks
- Do NOT change local gateway configurations during peak hours without approval

## Phase 3 — Rollback
- Restore previous routing table configurations
- Revert CoIP pool changes
- Re-establish VPC associations if broken
- Restore security group rules on local gateway
- Verify local network connectivity after changes
- Document local gateway changes and their impact

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
