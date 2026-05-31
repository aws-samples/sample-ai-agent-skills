---
title: "F3 — ElastiCache Security Group and Subnet Issues"
description: "Diagnose security group and subnet group misconfiguration preventing access to ElastiCache"
status: active
severity: HIGH
triggers:
  - "security group"
  - "subnet group"
  - "VPC"
  - "network access"
  - "port blocked"
  - "cannot reach"
owner: devops-agent
objective: "Resolve security group and subnet configuration issues to restore network access"
context: "ElastiCache clusters run inside a VPC and are not publicly accessible. Security groups control inbound traffic — port 6379 for Redis, 11211 for Memcached. The subnet group defines which subnets (and AZs) the cluster nodes are placed in. NACLs provide an additional layer of network filtering. Cross-VPC access requires VPC peering, Transit Gateway, or PrivateLink."
---

## Phase 1 — Triage

MUST:
- Check security groups: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].SecurityGroups'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissions'`
- Check subnet group: `aws elasticache describe-cache-subnet-groups --cache-subnet-group-name <subnet-group>`
- Verify client security group allows outbound to ElastiCache port
- Check subnet route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Check NACLs on ElastiCache subnets: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify client and ElastiCache are in the same VPC or have peering/TGW
- Check if the subnet has available IP addresses for new nodes
- Verify the VPC has DNS resolution enabled

MAY:
- Check VPC flow logs for rejected traffic
- Verify VPC peering route tables if cross-VPC
- Check Transit Gateway attachments if using TGW

## Phase 2 — Remediate

MUST:
- Add inbound rule to ElastiCache security group: port 6379 (Redis) or 11211 (Memcached) from client security group
- Ensure the subnet group includes subnets in the required AZs
- Verify NACLs allow traffic on the ElastiCache port (both inbound and outbound/ephemeral)

SHOULD:
- Use security group references (sg-xxx) instead of CIDR ranges for inbound rules
- Ensure subnet group has subnets in at least 2 AZs for Multi-AZ
- Document security group and subnet configuration

MAY:
- Set up VPC peering or Transit Gateway for cross-VPC access
- Implement VPC flow logs for network troubleshooting
- Use AWS PrivateLink for cross-account access

## Common Issues

- symptoms: "Connection timeout from application to ElastiCache"
  diagnosis: "Security group does not allow inbound on port 6379 from the application's security group."
  resolution: "Add inbound rule: Protocol=TCP, Port=6379, Source=<app-sg-id>."

- symptoms: "Cannot create cluster — insufficient IPs in subnet"
  diagnosis: "Subnet has no available IP addresses for ElastiCache nodes."
  resolution: "Use a subnet with more available IPs or add a new subnet to the subnet group."

- symptoms: "Cross-VPC application cannot connect"
  diagnosis: "No VPC peering or Transit Gateway between the VPCs. Route tables not updated."
  resolution: "Set up VPC peering or TGW. Update route tables in both VPCs. Update security groups."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add security group inbound rule | GREEN | Network configuration; allows specific traffic |
| Ensure subnet group has multi-AZ subnets | GREEN | Configuration verification; no immediate change |
| Fix NACL rules | YELLOW | Network change; affects all resources in the subnet |
| Set up VPC peering | YELLOW | Network infrastructure change; requires route table updates |
| Implement VPC flow logs | GREEN | Monitoring only; no operational impact |

## Escalation Conditions

- Security group change required on a shared security group used by multiple services
- Cross-VPC connectivity needed (VPC peering or Transit Gateway)
- NACL changes affecting other services in the same subnet
- Subnet has no available IP addresses for new nodes
- Security group rules conflict with compliance requirements

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-security-groups` | MEDIUM | Exposes network security rules and allowed sources |
| `describe-cache-subnet-groups` | LOW | Subnet configuration only |
| `describe-route-tables` | LOW | Network routing only |
| `describe-network-acls` | MEDIUM | Exposes network access control rules |
| VPC flow logs | MEDIUM | Exposes network traffic patterns |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix network issues
- NEVER suggest disabling AUTH to troubleshoot security group issues
- NEVER suggest disabling encryption in transit to simplify network configuration
- NEVER suggest reducing node count during peak traffic
- NEVER open security groups to 0.0.0.0/0 for ElastiCache access (not publicly accessible)

## Phase 3 — Rollback

If security group or subnet changes cause issues:
1. Revert security group inbound rules: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol tcp --port <port> --source-group <source-sg>`
2. If NACL rules were changed, revert to previous rules (note: NACLs are stateless — check both inbound and outbound)
3. If VPC peering was set up and causes routing issues, delete the peering connection and revert route tables
4. If subnet group was modified, revert to previous subnet configuration
5. Verify connectivity is restored after rollback

## Output Format

```yaml
root_cause: "security_network — <specific_cause>"
evidence:
  - type: security_groups
    content: "<inbound rules>"
  - type: subnet_group
    content: "<subnet configuration>"
  - type: route_tables
    content: "<routing configuration>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules or subnet configuration"
  long_term: "Document network architecture, implement VPC flow logs"
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"
