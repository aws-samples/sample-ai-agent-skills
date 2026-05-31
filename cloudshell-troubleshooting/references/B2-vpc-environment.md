---
title: "B2 — VPC Environment"
description: "Diagnose CloudShell VPC environment configuration issues"
status: active
severity: HIGH
triggers: ["VPC CloudShell", "cannot create VPC environment", "VPC environment failed", "private subnet CloudShell"]
owner: devops-agent
objective: "Resolve VPC-connected CloudShell environment issues"
context: "VPC environments require VPC, subnet, and security group. They run ENIs in the specified subnet. No internet by default."
---

## Phase 1 — Triage

MUST:
- Verify VPC exists and is available: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[*].{Id:VpcId,State:State,Cidr:CidrBlock}'`
- Verify subnet has available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{Id:SubnetId,AvailableIPs:AvailableIpAddressCount,Az:AvailabilityZone}'`
- Check security group exists and is in the correct VPC: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Verify IAM permissions include ec2:CreateNetworkInterface and ec2:DescribeNetworkInterfaces
- Check if subnet has route to NAT gateway for internet access: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"`

MAY:
- Review CloudTrail for VPC environment creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironment --max-results 10`
- Check ENI limits in the account: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-DF5E4CA3`

## Phase 2 — Remediate

MUST:
- Ensure subnet has sufficient available IP addresses (at least 1 per environment)
- Verify security group allows required outbound traffic for AWS API calls
- Grant ec2:CreateNetworkInterface, ec2:DeleteNetworkInterface, ec2:DescribeNetworkInterfaces permissions

SHOULD:
- Use private subnets with NAT gateway for internet-requiring workloads
- Add VPC endpoints for S3, STS, and other frequently used services

MAY:
- Create a dedicated subnet for CloudShell VPC environments
- Set up CloudShell-specific security group with minimal required rules

## Safety Ratings

```
safety_ratings:
  - "describe-vpcs / describe-subnets: GREEN — read-only VPC inspection"
  - "create-vpc-endpoint: YELLOW — creates billable resource"
  - "authorize-security-group-egress: YELLOW — modifies network rules"
  - "create-nat-gateway: YELLOW — creates billable resource, modifies routing"
```

## Escalation Conditions

- Subnet has no available IP addresses and cannot be expanded
- VPC architecture does not support NAT gateway addition
- ENI quota reached and increase requires AWS support
- Security group changes conflict with organization network policies

## Data Sensitivity

- `aws ec2 describe-vpcs`: LOW — VPC metadata
- `aws ec2 describe-subnets`: LOW — subnet configuration
- `aws ec2 describe-security-groups`: MEDIUM — network access rules
- `aws cloudtrail lookup-events`: MEDIUM — user activity records

## Prohibited Actions

- NEVER create CloudShell VPC environments in production VPCs without approval
- NEVER assign public IP addresses to CloudShell ENIs
- NEVER use default security group for CloudShell VPC environments
- NEVER modify existing security groups shared with other resources

## Phase 3 — Rollback

1. If VPC environment was created in wrong VPC: delete environment and recreate in correct VPC
2. If security group was modified: revert rules to original configuration
3. If VPC endpoint was created: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <id>`
4. Clean up orphaned ENIs: `aws ec2 describe-network-interfaces --filters "Name=description,Values=*CloudShell*" "Name=status,Values=available"`
5. Document changes and notify network team

## Common Issues

- symptoms: "CreateEnvironment fails with 'Subnet has insufficient IP addresses'"
  diagnosis: "The specified subnet has no available IP addresses for the CloudShell ENI."
  resolution: "Use a subnet with available IPs or expand the subnet CIDR. Each VPC environment requires one IP address."

## Output Format

```yaml
root_cause: "vpc_environment — <specific_cause>"
evidence:
  - type: vpc_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC configuration"
  long_term: "Dedicate subnet and security group for CloudShell VPC environments"
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
