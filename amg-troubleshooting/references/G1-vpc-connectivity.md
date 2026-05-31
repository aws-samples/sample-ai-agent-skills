---
title: "G1 — VPC Connectivity Issues"
description: "Diagnose VPC connectivity problems with Amazon Managed Grafana workspaces"
status: active
severity: HIGH
triggers:
  - "VPC connectivity"
  - "VPC configuration"
  - "cannot reach VPC"
  - "network error"
  - "connection refused"
owner: devops-agent
objective: "Identify and resolve VPC connectivity issues preventing the AMG workspace from reaching private resources"
context: "AMG workspaces can be configured with VPC connectivity to reach private resources (databases, self-managed Prometheus, OpenSearch in VPC). VPC configuration requires specifying subnets and security groups. Connectivity failures typically stem from incorrect subnet configuration, security group rules, route table issues, or missing VPC endpoints."
---

## Phase 1 — Triage

MUST:
- Check workspace VPC configuration: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.vpcConfiguration'`
- Verify the subnets exist and are in the correct VPC: `aws ec2 describe-subnets --subnet-ids <subnet-ids>`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify the security groups allow outbound traffic to the target resource

SHOULD:
- Check route tables for the configured subnets: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify the target resource's security group allows inbound traffic from the workspace security group
- Check if NAT Gateway is needed for internet access from private subnets

MAY:
- Check VPC flow logs for denied traffic: `aws ec2 describe-flow-logs --filter Name=resource-id,Values=<vpc-id>`
- Verify DNS resolution works within the VPC
- Check network ACLs for the subnets

## Phase 2 — Remediate

MUST:
- Configure VPC on the workspace if not set: `aws grafana update-workspace --workspace-id <workspace-id> --vpc-configuration subnetIds=<subnet-ids>,securityGroupIds=<sg-ids>`
- Ensure security groups allow outbound HTTPS (443) and the target resource port
- Ensure the target resource's security group allows inbound from the workspace security group

SHOULD:
- Use subnets in at least two Availability Zones for high availability
- Configure a NAT Gateway if the workspace needs both VPC access and internet access
- Use VPC endpoints for AWS services to avoid routing through the internet

MAY:
- Set up VPC peering or Transit Gateway for cross-VPC connectivity
- Implement network monitoring for ongoing connectivity verification

## Common Issues

- symptoms: "Workspace cannot reach private data source after VPC configuration"
  diagnosis: "Security group rules do not allow traffic between the workspace and the target resource."
  resolution: "Add inbound rules on the target resource's security group allowing traffic from the workspace security group on the required port."

- symptoms: "VPC configuration update fails"
  diagnosis: "Specified subnets are not in the same VPC, or security groups are in a different VPC."
  resolution: "Ensure all subnets and security groups are in the same VPC. Use subnets in at least two AZs."

- symptoms: "Workspace loses internet access after VPC configuration"
  diagnosis: "VPC subnets are private without a NAT Gateway, so the workspace cannot reach public endpoints."
  resolution: "Add a NAT Gateway to the VPC and update route tables, or use VPC endpoints for AWS services."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace (vpcConfiguration): GREEN — read-only VPC check"
  - "describe-subnets: GREEN — read-only subnet check"
  - "describe-security-groups: GREEN — read-only SG check"
  - "update-workspace (vpc-configuration): YELLOW — creates ENIs, changes network topology"
  - "modify security group rules: YELLOW — changes network access controls"
```

## Escalation Conditions

- VPC configuration changes require network team approval
- Security group modifications affect other resources in the VPC
- NAT Gateway setup needed for internet access from private subnets
- VPC peering or Transit Gateway required for cross-VPC data sources

## Data Sensitivity

- VPC configuration: MEDIUM — reveals network architecture and subnet layout
- Security group rules: MEDIUM — reveals network access controls
- Route tables: MEDIUM — reveals network routing topology
- VPC flow logs: HIGH — contain network traffic metadata

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER configure VPC with public subnets unless explicitly required
- NEVER open security groups to 0.0.0.0/0 for troubleshooting

## Phase 3 — Rollback

1. If VPC configuration was added incorrectly, remove it: `aws grafana update-workspace --workspace-id <id> --remove-vpc-configuration`
2. If security group rules were changed, restore original rules
3. If NAT Gateway was created, delete it to stop incurring costs (after confirming it's not needed)
4. Verify workspace connectivity to all data sources after rollback
5. Document network changes and coordinate with the network team

## Output Format

```yaml
root_cause: "vpc_connectivity — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<workspace VPC configuration>"
  - type: security_groups
    content: "<security group rules>"
  - type: subnets
    content: "<subnet configuration and route tables>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules, subnet configuration, or routing"
  long_term: "Document VPC architecture and automate network configuration with IaC"
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
