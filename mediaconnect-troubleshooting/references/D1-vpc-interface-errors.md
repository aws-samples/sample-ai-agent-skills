---
title: "D1 — VPC Interface Errors"
description: "Diagnose MediaConnect VPC interface creation and connectivity issues"
status: active
severity: HIGH
triggers: ["VPC interface error", "ENI creation failed", "VPC flow not working", "private flow", "AddFlowVpcInterfaces failed"]
owner: devops-agent
objective: "Resolve VPC interface configuration and connectivity issues for private MediaConnect flows"
context: "VPC interfaces create ENIs in customer VPCs for private source/output connectivity. Requires subnet, security group, and IAM role configuration."
---

## Phase 1 — Triage

MUST:
- Check VPC interfaces on the flow: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.VpcInterfaces[*].{Name:Name,SubnetId:SubnetId,SecurityGroupIds:SecurityGroupIds,NetworkInterfaceIds:NetworkInterfaceIds}'`
- Verify security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].{GroupId:GroupId,Ingress:IpPermissions,Egress:IpPermissionsEgress}'`
- Check subnet has available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{SubnetId:SubnetId,AvailableIpAddressCount:AvailableIpAddressCount,AvailabilityZone:AvailabilityZone}'`

SHOULD:
- Verify the IAM role has ec2:CreateNetworkInterface, ec2:DescribeNetworkInterfaces permissions
- Check ENI limits for the account: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-DF5E4CA3`
- Verify network ACLs on the subnet

MAY:
- Check VPC Flow Logs for traffic analysis
- Verify route table for the subnet

## Phase 2 — Remediate

MUST:
- Fix security group inbound rules: `aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol udp --port <port> --cidr <source-cidr>`
- Ensure subnet has available IPs and is in the correct AZ
- Fix IAM role permissions for ENI management

SHOULD:
- Add VPC interface if missing: `aws mediaconnect add-flow-vpc-interfaces --flow-arn <flow-arn> --vpc-interfaces Name=<name>,SubnetId=<subnet-id>,SecurityGroupIds=<sg-id>,RoleArn=<role-arn>`
- Verify network ACLs allow both inbound and outbound on the protocol port

MAY:
- Enable VPC Flow Logs for ongoing monitoring
- Create dedicated security group for MediaConnect VPC interfaces

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow inspection"
  - "describe-security-groups: GREEN — read-only security group inspection"
  - "authorize-security-group-ingress: YELLOW — opens network access, verify CIDR is specific"
  - "add-flow-vpc-interfaces: YELLOW — creates ENI in VPC, consumes IP address"
```

## Escalation Conditions

- ENI creation fails due to account-level ENI limits
- VPC interface created but no traffic flows despite correct security group and NACL
- Subnet has no available IP addresses
- IAM role trust policy does not allow MediaConnect to assume it

## Data Sensitivity

- `aws ec2 describe-security-groups`: MEDIUM — reveals network access rules
- `aws ec2 describe-subnets`: LOW — returns subnet metadata
- `aws ec2 authorize-security-group-ingress`: HIGH — modifies network access controls
- `aws mediaconnect add-flow-vpc-interfaces`: MEDIUM — creates ENI with network access

## Prohibited Actions

- NEVER open security group to 0.0.0.0/0 for MediaConnect VPC interfaces
- NEVER create VPC interfaces in public subnets unless explicitly required
- NEVER delete ENIs manually — use remove-flow-vpc-interfaces to clean up
- NEVER modify network ACLs without understanding impact on other resources in the subnet

## Phase 3 — Rollback

1. If VPC interface was added incorrectly, remove it: `aws mediaconnect remove-flow-vpc-interfaces --flow-arn <flow-arn> --vpc-interface-names <name>`
2. If security group was opened too broadly, revoke: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol udp --port <port> --cidr <overly-broad-cidr>`
3. Verify ENI was released after VPC interface removal: `aws ec2 describe-network-interfaces --filters Name=description,Values="*MediaConnect*"`
4. If ENI is orphaned, delete it: `aws ec2 delete-network-interface --network-interface-id <eni-id>` (only after confirming it is not in use)
5. Document the VPC interface change and notify the network operations team

## Common Issues

- symptoms: "AddFlowVpcInterfaces returns error about insufficient IP addresses"
  diagnosis: "Subnet has no available IP addresses for the ENI."
  resolution: "Use a subnet with available IPs or expand the subnet CIDR. Check with describe-subnets."

- symptoms: "VPC interface created but source receives no traffic"
  diagnosis: "Security group does not allow inbound on the protocol port from the source IP."
  resolution: "Add inbound rule for the protocol (UDP) and port from the source CIDR."

- symptoms: "AddFlowVpcInterfaces returns AccessDeniedException"
  diagnosis: "IAM role lacks ec2:CreateNetworkInterface or ec2:DescribeNetworkInterfaces permissions."
  resolution: "Update the IAM role policy to include ec2:CreateNetworkInterface, ec2:DescribeNetworkInterfaces, ec2:CreateNetworkInterfacePermission."

## Output Format

```yaml
root_cause: "vpc_interface — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<security group rules, subnet details, ENI status>"
severity: HIGH
mitigation:
  immediate: "Fix VPC interface configuration"
  long_term: "Use IaC to manage VPC interfaces with validated security groups"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
