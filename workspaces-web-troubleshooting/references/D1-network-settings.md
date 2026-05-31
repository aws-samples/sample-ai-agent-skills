---
title: "D1 — Network Settings"
description: "Diagnose WorkSpaces Web network settings configuration issues"
status: active
severity: HIGH
triggers: ["network settings error", "cannot create network settings", "session network failure"]
owner: devops-agent
objective: "Resolve network settings configuration issues for WorkSpaces Web portals"
context: "Network settings define the VPC, subnets, and security groups through which browser sessions access resources. Sessions route all traffic through this VPC configuration."
---

## Phase 1 — Triage

MUST:
- Get network settings details: `aws workspaces-web get-network-settings --network-settings-arn <arn>`
- Verify VPC exists and is available: `aws ec2 describe-vpcs --vpc-ids <vpc-id>`
- Verify subnets exist and have available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,AvailableIps:AvailableIpAddressCount,State:State}'`

SHOULD:
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify subnets span at least two availability zones
- Check route tables for internet access (NAT gateway or IGW)

MAY:
- Verify VPC DNS settings are enabled: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Check for VPC flow logs to diagnose traffic issues

## Phase 2 — Remediate

MUST:
- Create network settings with valid VPC, subnets, and security groups: `aws workspaces-web create-network-settings --vpc-id <vpc-id> --subnet-ids <subnet-1> <subnet-2> --security-group-ids <sg-id>`
- Ensure subnets are in at least two availability zones
- Ensure security groups allow outbound HTTPS (443) traffic

SHOULD:
- Configure NAT gateway for private subnets that need internet access
- Enable VPC DNS support and DNS hostnames
- Use dedicated security groups for WorkSpaces Web (not shared with other workloads)

MAY:
- Configure VPC endpoints for AWS service access without internet
- Set up VPC flow logs for network troubleshooting
- Create separate network settings for different portal use cases

## Safety Ratings

```
safety_ratings:
  - "get-network-settings: GREEN — read-only network configuration"
  - "describe-vpcs: GREEN — read-only VPC details"
  - "describe-subnets: GREEN — read-only subnet details"
  - "create-network-settings: YELLOW — creates VPC association resource"
  - "associate-network-settings: YELLOW — changes portal network path"
```

## Escalation Conditions

- VPC does not have sufficient IP addresses in the required subnets
- Network settings creation fails with internal service error
- Changing network settings requires VPC modifications that affect other workloads
- Cross-account VPC access is required for session connectivity

## Data Sensitivity

- `aws workspaces-web get-network-settings`: MEDIUM — reveals VPC and subnet configuration
- `aws ec2 describe-vpcs`: LOW — returns VPC metadata
- `aws ec2 describe-subnets`: LOW — returns subnet metadata
- `aws ec2 describe-security-groups`: MEDIUM — reveals security rules

## Prohibited Actions

- NEVER use the default VPC for production portal network settings
- NEVER create security groups with 0.0.0.0/0 inbound rules for WorkSpaces Web
- NEVER modify shared VPC resources without coordinating with the network team
- NEVER delete network settings that are associated with an active portal

## Phase 3 — Rollback

1. Disassociate network settings from the portal if changes caused issues: `aws workspaces-web disassociate-network-settings --portal-arn <portal-arn>`
2. Re-associate previous network settings
3. Delete incorrectly created network settings: `aws workspaces-web delete-network-settings --network-settings-arn <arn>`
4. Revert any VPC, subnet, or security group changes made during troubleshooting
5. Verify portal sessions work correctly after rollback

## Common Issues

- symptoms: "Network settings creation fails with 'subnets must be in different AZs'"
  diagnosis: "All provided subnets are in the same availability zone."
  resolution: "Provide subnets from at least two different availability zones."

## Output Format

```yaml
root_cause: "network_settings — <specific_cause>"
evidence:
  - type: network_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix network settings configuration"
  long_term: "Document network architecture and use IaC for network settings"
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
