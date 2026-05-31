---
title: "A2 — Subnet Creation Failures"
description: "Diagnose subnet creation failures in Local Zones"
status: active
severity: HIGH
triggers: ["cannot create subnet", "subnet creation failed", "no subnet in Local Zone", "CIDR error"]
owner: devops-agent
objective: "Resolve subnet creation issues in Local Zones"
context: "Local Zone subnets are created within existing VPCs. The subnet CIDR must be within the VPC CIDR range. The Local Zone must be opted in before subnet creation."
---

## Phase 1 — Triage

MUST:
- Verify Local Zone is opted in: `aws ec2 describe-availability-zones --zone-names <local-zone-name>`
- Check VPC CIDR range: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[0].CidrBlockAssociationSet'`
- List existing subnets to check for CIDR conflicts: `aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" --query 'Subnets[*].{SubnetId:SubnetId,CidrBlock:CidrBlock,AZ:AvailabilityZone}'`

SHOULD:
- Verify the desired CIDR does not overlap with existing subnets
- Check subnet quota for the VPC

MAY:
- Review CloudTrail for CreateSubnet errors
- Check if VPC needs additional CIDR blocks for Local Zone subnets

## Phase 2 — Remediate

MUST:
- Opt in to Local Zone if not already enabled
- Choose a non-overlapping CIDR within the VPC range
- Create subnet: `aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block <cidr> --availability-zone <local-zone-name>`

SHOULD:
- Associate subnet with appropriate route table
- Configure network ACLs for the new subnet
- Tag subnet for identification

MAY:
- Add secondary CIDR to VPC if address space is exhausted
- Create subnets in multiple Local Zones for geographic coverage

## Safety Ratings

```
safety_ratings:
  - "describe-availability-zones: GREEN — read-only zone check"
  - "describe-vpcs: GREEN — read-only VPC details"
  - "describe-subnets: GREEN — read-only subnet enumeration"
  - "create-subnet: YELLOW — creates new network resource in VPC"
  - "associate-vpc-cidr-block: YELLOW — expands VPC address space"
```

## Escalation Conditions

- VPC CIDR exhausted and secondary CIDR addition requires network team approval
- Subnet creation fails with internal errors after fixing all parameters
- Network ACL or route table changes require security team review
- Multiple Local Zone subnets needed across accounts requiring coordinated CIDR planning

## Data Sensitivity

- VPC CIDR information: MEDIUM — reveals network architecture
- Subnet configuration: MEDIUM — reveals network segmentation
- Route table associations: MEDIUM — reveals traffic routing patterns
- CloudTrail events: MEDIUM — reveals IAM principals and actions

## Prohibited Actions

- NEVER create subnets with overlapping CIDRs in the same VPC
- NEVER use the entire VPC CIDR for a single Local Zone subnet
- NEVER create Local Zone subnets without proper route table association
- NEVER delete existing subnets to free CIDR space without migrating resources

## Phase 3 — Rollback

1. If subnet was created with wrong CIDR, delete it (only if no resources attached): `aws ec2 delete-subnet --subnet-id <subnet-id>`
2. If secondary CIDR was added unnecessarily, disassociate it: `aws ec2 disassociate-vpc-cidr-block --association-id <assoc-id>`
3. If route table association was incorrect, replace with correct route table
4. Verify VPC routing is intact after rollback

## Common Issues

- symptoms: "CreateSubnet returns InvalidParameterValue for availability zone"
  diagnosis: "Local Zone is not opted in or zone name is incorrect."
  resolution: "Opt in to the Local Zone first. Verify zone name with describe-availability-zones --all-availability-zones."

- symptoms: "CreateSubnet returns InvalidSubnet.Conflict"
  diagnosis: "Requested CIDR overlaps with an existing subnet in the VPC."
  resolution: "Choose a non-overlapping CIDR. List existing subnets to find available ranges."

## Output Format

```yaml
root_cause: "subnet_creation — <specific_cause>"
evidence:
  - type: subnet_error
    content: "<error details and VPC CIDR information>"
severity: HIGH
mitigation:
  immediate: "Fix CIDR or opt-in issue"
  long_term: "Implement CIDR planning for Local Zone expansion"
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
