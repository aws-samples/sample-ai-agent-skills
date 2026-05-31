---
title: "F1 — Integration with VPC"
description: "Diagnose VPC integration issues with Private 5G"
status: active
severity: HIGH
triggers: ["VPC connectivity failed", "ENI error", "subnet issue", "routing problem", "security group blocking"]
owner: devops-agent
objective: "Resolve VPC integration issues for Private 5G network traffic"
context: "Private 5G routes device traffic through ENIs in your VPC subnet. Proper subnet, security group, and route table configuration is required."
---

## Phase 1 — Triage

MUST:
- Get network details including VPC configuration: `aws private-networks get-network --network-arn <arn>`
- Check ENIs associated with Private 5G: `aws ec2 describe-network-interfaces --filters Name=description,Values="*private-networks*"`
- Verify security group rules: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check subnet available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{SubnetId:SubnetId,AvailableIPs:AvailableIpAddressCount,CidrBlock:CidrBlock}'`
- Review route table: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check VPC flow logs for dropped traffic: `aws logs filter-log-events --log-group-name <vpc-flow-log-group> --filter-pattern "REJECT"`

MAY:
- Verify NACLs are not blocking traffic: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Check if VPC DNS resolution is enabled

## Phase 2 — Remediate

MUST:
- Ensure subnet has available IP addresses for ENIs
- Update security group to allow Private 5G device traffic
- Verify route table has routes for device traffic destinations

SHOULD:
- Add specific security group rules rather than broad allow-all rules
- Ensure NACL rules allow bidirectional traffic for Private 5G
- Configure VPC DNS for device name resolution

MAY:
- Set up VPC flow logs for ongoing monitoring
- Create dedicated security group for Private 5G ENIs
- Configure VPC endpoints for AWS service access from devices

## Safety Ratings

```
safety_ratings:
  - "describe-network-interfaces: GREEN — read-only ENI listing"
  - "describe-security-groups: GREEN — read-only security review"
  - "describe-subnets: GREEN — read-only subnet details"
  - "authorize-security-group-ingress: YELLOW — modifies network access rules"
```

## Escalation Conditions

- ENIs are missing or in error state
- Subnet has zero available IP addresses
- Security group changes do not resolve connectivity
- VPC flow logs show persistent REJECT entries for Private 5G traffic

## Data Sensitivity

- `aws ec2 describe-network-interfaces`: MEDIUM — reveals network topology
- `aws ec2 describe-security-groups`: MEDIUM — reveals access control rules
- VPC flow logs: MEDIUM — reveals traffic patterns and IP addresses
- Subnet configuration: MEDIUM — reveals network architecture

## Prohibited Actions

- NEVER delete Private 5G ENIs manually
- NEVER open security groups to 0.0.0.0/0 for troubleshooting
- NEVER modify the VPC CIDR block to fix IP exhaustion
- NEVER change subnet associations without understanding impact on the Private 5G network

## Phase 3 — Rollback

1. If security group rules were modified, restore original rules using saved configuration
2. If NACL rules were changed, restore original entries
3. If route table was modified, restore original routes
4. Verify Private 5G device traffic flows correctly after rollback
5. Confirm other VPC resources are unaffected by changes

## Common Issues

- symptoms: "Devices connect to Private 5G but cannot reach internet or AWS services"
  diagnosis: "Route table missing default route or NAT gateway route."
  resolution: "Add route to NAT gateway or internet gateway in the subnet route table."

- symptoms: "Devices cannot reach resources in other VPC subnets"
  diagnosis: "Security group or NACL blocking cross-subnet traffic."
  resolution: "Update security group to allow traffic to destination subnet CIDRs."

## Output Format

```yaml
root_cause: "vpc_integration — <specific_cause>"
evidence:
  - type: vpc_configuration
    content: "<VPC details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC integration issue"
  long_term: "Document VPC configuration requirements for Private 5G"
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
