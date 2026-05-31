---
title: "C1 — Network Connectivity"
description: "Diagnose network connectivity issues on Outposts Servers"
status: active
severity: HIGH
triggers: ["cannot reach instance on outpost", "network timeout", "CoIP not working", "local gateway issues", "on-premises connectivity"]
owner: devops-agent
objective: "Resolve network connectivity issues between Outposts Server instances and on-premises or Region networks"
context: "Outposts Servers use local gateways (LGW) for on-premises connectivity and service links for Region connectivity. Customer-owned IP (CoIP) pools provide local addressing."
---

## Phase 1 — Triage

MUST:
- Check local gateway route table: `aws ec2 describe-local-gateway-route-tables --query 'LocalGatewayRouteTables[*].{Id:LocalGatewayRouteTableId,LgwId:LocalGatewayId,OutpostArn:OutpostArn,State:State}'`
- Verify CoIP pool configuration: `aws ec2 describe-coip-pools --query 'CoipPools[*].{PoolId:PoolId,PoolCidrs:PoolCidrs,LocalGatewayRouteTableId:LocalGatewayRouteTableId}'`
- Check VPC route table for Outpost subnet: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<outpost-subnet-id>" --query 'RouteTables[*].Routes'`

SHOULD:
- Verify local gateway VIF group associations: `aws ec2 describe-local-gateway-virtual-interface-groups --query 'LocalGatewayVirtualInterfaceGroups[*].{Id:LocalGatewayVirtualInterfaceGroupId,LgwId:LocalGatewayId}'`
- Check security group rules for Outpost instances
- Verify network ACLs on the Outpost subnet

MAY:
- Test connectivity from on-premises to Outpost instances
- Review VPC flow logs for the Outpost subnet

## Phase 2 — Remediate

MUST:
- Ensure CoIP pool is associated with the local gateway route table
- Verify local gateway route table has correct routes for on-premises networks
- Confirm VPC route table includes routes for CoIP address ranges

SHOULD:
- Verify security groups allow required traffic from on-premises CIDR ranges
- Ensure network ACLs permit bidirectional traffic

MAY:
- Configure VPC flow logs for the Outpost subnet for ongoing monitoring
- Implement network monitoring for CoIP pool utilization

## Common Issues

- symptoms: "Instances on Outpost cannot communicate with on-premises network"
  diagnosis: "CoIP pool not associated with local gateway route table or missing routes."
  resolution: "Associate CoIP pool with LGW route table. Add routes for on-premises CIDRs to the LGW route table."

## Safety Ratings

```
safety_ratings:
  - "describe-local-gateway-route-tables: GREEN — read-only enumeration"
  - "describe-coip-pools: GREEN — read-only pool check"
  - "Route table modifications: RED — can disrupt all network connectivity on the Outpost"
  - "Security group changes: YELLOW — modifies network access controls"
```

## Escalation Conditions

- Complete loss of network connectivity to all instances on the Outpost
- Local gateway shows error state or is not responding
- CoIP pool exhaustion preventing new instance connectivity
- Network issues affecting the service link (see C2 runbook)

## Data Sensitivity

- `aws ec2 describe-local-gateway-route-tables`: MEDIUM — reveals network topology
- `aws ec2 describe-coip-pools`: MEDIUM — reveals IP addressing scheme
- VPC flow logs: HIGH — contain source/destination IPs and traffic patterns
- On-premises network details: HIGH — reveals corporate network topology

## Prohibited Actions

- NEVER delete or modify the local gateway route table without documenting current state
- NEVER remove CoIP pool associations without understanding the impact on running instances
- NEVER modify VPC route tables for Outpost subnets without a rollback plan
- NEVER expose Outpost instance details or CoIP ranges in public documentation

## Phase 3 — Rollback

1. If route table entries were modified, revert to documented previous state
2. If CoIP associations were changed, restore original associations
3. If security groups were modified, revert to previous rules: `aws ec2 describe-security-group-rules --security-group-id <sg-id>`
4. Verify all instances regain connectivity after rollback
5. Document network configuration changes and their effects

## Output Format

```yaml
root_cause: "network_connectivity — <specific_cause>"
evidence:
  - type: network_config
    content: "<route table, CoIP, and LGW details>"
severity: HIGH
mitigation:
  immediate: "Restore network connectivity"
  long_term: "Document network topology and implement monitoring"
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
