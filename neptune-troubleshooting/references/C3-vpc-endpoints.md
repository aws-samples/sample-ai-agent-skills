---
title: "C3 — VPC/Endpoints"
description: "Diagnose Neptune VPC configuration and endpoint issues"
status: active
severity: MEDIUM
triggers:
  - "vpc endpoint"
  - "private link"
  - "cross vpc"
  - "endpoint access"
owner: devops-agent
objective: "Identify and resolve Neptune VPC and endpoint configuration issues"
context: "Neptune is VPC-only. Cross-VPC access requires VPC peering, Transit Gateway, or PrivateLink. Custom endpoints can route traffic to specific instances."
---

## Phase 1 — Triage

MUST:
- Check cluster endpoints: `aws neptune describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>`
- Verify subnet group: `aws neptune describe-db-subnet-groups --db-subnet-group-name <subnet-group>`
- Check VPC configuration: `aws ec2 describe-vpcs --vpc-ids <vpc-id>`
- Verify security groups: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check VPC peering connections: `aws ec2 describe-vpc-peering-connections --filters Name=requester-vpc-info.vpc-id,Values=<vpc-id>`
- Verify route tables for cross-VPC routing: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`
- Check DNS resolution from client VPC
- Verify network ACLs allow traffic

MAY:
- Check Transit Gateway attachments: `aws ec2 describe-transit-gateway-attachments`
- Verify custom endpoints configuration
- Test connectivity from different subnets

## Phase 2 — Remediate

MUST:
- Ensure network path exists between client and Neptune VPC
- Fix security group and network ACL rules
- Verify DNS resolution works across VPCs

SHOULD:
- Use custom endpoints for workload isolation
- Configure reader endpoint for read-heavy workloads
- Document network topology

MAY:
- Set up VPC flow logs for network debugging
- Implement network monitoring

## Common Issues

- symptoms: "Cannot connect from peered VPC"
  diagnosis: "Route tables or DNS resolution not configured for VPC peering."
  resolution: "Add routes for peered VPC CIDR. Enable DNS resolution on peering connection."

- symptoms: "Custom endpoint not routing correctly"
  diagnosis: "Custom endpoint member list does not include desired instances."
  resolution: "Update custom endpoint to include correct instance members."

- symptoms: "Intermittent connectivity from different subnets"
  diagnosis: "Network ACLs or route tables differ between subnets."
  resolution: "Verify consistent network ACL and route table configuration across subnets."

## Output Format

```yaml
root_cause: "vpc_endpoint — <specific_cause>"
evidence:
  - type: endpoints
    content: "<cluster endpoint configuration>"
  - type: network
    content: "<VPC, subnet, routing configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix network configuration"
  long_term: "Document and automate network setup"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — VPC endpoint configuration is infrastructure-level
- Service Disruption Risk: MEDIUM — endpoint issues affect connectivity to AWS services
- Compliance Impact: MEDIUM — VPC endpoints enforce private connectivity requirements

## Escalation Conditions
- VPC endpoint becomes unavailable affecting Neptune operations
- Endpoint policy changes block required AWS service access
- Cross-AZ endpoint routing causes latency issues
- Endpoint security group changes break connectivity
- Multiple VPC endpoints fail simultaneously

## Data Sensitivity
- VPC endpoint IDs and configurations — INTERNAL
- Endpoint policies — INTERNAL
- Security group rules — INTERNAL
- Route table configurations — INTERNAL
- Never expose VPC internal architecture details externally

## Prohibited Actions
- Do NOT delete VPC endpoints without verifying Neptune does not depend on them
- Do NOT apply overly restrictive endpoint policies without testing
- Do NOT modify endpoint security groups during production hours without approval
- Do NOT create VPC endpoints in incorrect subnets
- Do NOT ignore endpoint state changes
- Do NOT bypass VPC endpoint routing for troubleshooting

## Phase 3 — Rollback
- Restore previous VPC endpoint policies
- Revert security group changes on VPC endpoints
- Re-create deleted VPC endpoints if needed
- Verify Neptune connectivity through VPC endpoints after changes
- Restore route table entries if modified
- Document VPC endpoint changes and their impact

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
