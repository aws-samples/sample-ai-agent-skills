---
title: "H2 — VPC/Subnet Configuration"
description: "Diagnose VPC and subnet configuration issues for DMS"
status: active
severity: HIGH
triggers:
  - "VPC error"
  - "subnet error"
  - "network unreachable"
  - "replication subnet group"
  - "cannot reach endpoint"
owner: devops-agent
objective: "Fix VPC and subnet configuration for DMS replication instances"
context: "DMS replication instances run in a VPC and require proper subnet group configuration, security groups, route tables, and network connectivity to both source and target endpoints. Misconfigured networking is a common cause of endpoint connectivity failures."
---

## Phase 1 — Triage

MUST:
- Check replication subnet group: `aws dms describe-replication-subnet-groups --filters Name=replication-subnet-group-id,Values=<group-id>`
- Check replication instance VPC and security groups: `aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id> --query 'ReplicationInstances[*].{VPC:ReplicationSubnetGroup.VpcId,Subnets:ReplicationSubnetGroup.Subnets[*].SubnetIdentifier,SGs:VpcSecurityGroups}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check subnet route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Verify subnets are in at least 2 AZs (required for subnet groups)
- Check if public IP is needed for public endpoint access
- Verify VPC peering or transit gateway for cross-VPC endpoints
- Check NACLs on the subnets: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

MAY:
- Check VPC DNS settings for endpoint name resolution
- Verify NAT gateway for private subnet internet access
- Test connectivity from an EC2 instance in the same subnet

## Phase 2 — Remediate

MUST:
- Ensure subnet group has subnets in at least 2 AZs
- Configure security groups to allow outbound to source and target ports
- Ensure route tables have routes to source and target networks

SHOULD:
- For public endpoints: use public subnet with internet gateway and enable public IP
- For private endpoints: ensure VPC peering, transit gateway, or VPN connectivity
- Create replication subnet group: `aws dms create-replication-subnet-group --replication-subnet-group-identifier <id> --replication-subnet-group-description <desc> --subnet-ids <subnet-ids>`

MAY:
- Set up VPC Flow Logs for network troubleshooting
- Use VPC endpoints for AWS service targets (S3, Redshift)

## Common Issues

- symptoms: "Replication subnet group requires subnets in at least 2 AZs"
  diagnosis: "Subnet group has subnets in only one AZ."
  resolution: "Add subnets from at least 2 AZs to the replication subnet group."

- symptoms: "Cannot reach source endpoint from replication instance"
  diagnosis: "Security group, route table, or NACL blocking traffic."
  resolution: "Allow outbound traffic to source IP:port. Add route to source network."

- symptoms: "Public endpoint unreachable"
  diagnosis: "Replication instance in private subnet or no public IP."
  resolution: "Use public subnet with internet gateway. Enable publicly accessible."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure subnet group has subnets in 2+ AZs | YELLOW | Subnet group change — may require instance modification |
| Configure security groups for outbound access | YELLOW | Network change — verify no unintended access |
| Ensure route tables have routes to endpoints | YELLOW | Routing change — affects all resources in subnet |
| Create replication subnet group | GREEN | New resource creation — non-destructive |
| Set up VPC Flow Logs | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- VPC/subnet changes affect other services in the same network
- Network configuration is managed by a separate networking team

## Data Sensitivity

- **Classification: HIGH**
- Subnet and VPC configuration reveals network architecture
- Security group rules expose network access boundaries for replication
- Route tables reveal connectivity to source and target databases
- NACL rules expose network-level access controls

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest opening all ports in security groups to resolve connectivity
- **NEVER** suggest placing replication instances in public subnets with public IPs for production

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified subnet group | Revert subnet group to previous configuration |
| Changed security group rules | Revert security group rules to previous state |
| Modified route tables | Revert route table entries to previous configuration |
| Created VPC endpoints | Delete VPC endpoints if not needed |

## Output Format

```yaml
root_cause: "vpc_subnet — <specific_cause>"
evidence:
  - type: subnet_group
    content: "<subnet group configuration>"
  - type: security_groups
    content: "<security group rules>"
  - type: route_tables
    content: "<route table entries>"
severity: HIGH
mitigation:
  immediate: "Fix VPC/subnet configuration"
  long_term: "Document network requirements and automate with IaC"
```

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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
