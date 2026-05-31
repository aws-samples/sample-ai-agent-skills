---
title: "I1 — Subnet IP Exhaustion"
description: "Diagnose failures caused by insufficient available IP addresses in subnets"
status: active
severity: CRITICAL
triggers:
  - "InsufficientFreeAddressesInSubnet"
  - "Cannot launch instance"
  - "ENI creation failed"
owner: devops-agent
objective: "Resolve IP exhaustion and restore resource provisioning"
context: "Each subnet has a fixed CIDR. 5 IPs are reserved (first 4 + last). ENIs, Lambda VPC, ELB, NAT gateway, and other services consume IPs. A /24 has 251 usable IPs. Exhaustion prevents new resource creation."
---

## Phase 1 — Triage

MUST:
- Check available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-id>` → AvailableIpAddressCount
- Identify what's consuming IPs: ENIs in the subnet
- List ENIs: `aws ec2 describe-network-interfaces --filters Name=subnet-id,Values=<subnet-id>`
- Check if Lambda, ELB, or other managed services are consuming IPs

SHOULD:
- Check if unused ENIs can be cleaned up
- Verify if the subnet CIDR is appropriately sized for the workload
- Check if secondary CIDRs can be added to the VPC

MAY:
- Analyze ENI ownership to identify the largest consumers
- Check if VPC CNI (EKS) prefix delegation could reduce IP consumption

## Common Issues

- symptoms: "Can't launch new instances, AvailableIpAddressCount is 0"
  diagnosis: "Subnet CIDR exhausted by existing resources."
  resolution: "Clean up unused ENIs, move resources to a larger subnet, or add secondary CIDR."

- symptoms: "Lambda functions failing in VPC, IP exhaustion"
  diagnosis: "Lambda Hyperplane ENIs consume IPs. High concurrency can exhaust small subnets."
  resolution: "Use larger subnets (/24 or bigger) for Lambda VPC functions."

## Safety Ratings

```
safety_ratings:
  - "describe-subnets: GREEN — read-only subnet and available IP check"
  - "describe-network-interfaces: GREEN — read-only ENI listing"
  - "describe-vpcs: GREEN — read-only VPC CIDR inspection"
  - "delete-network-interface (clean up unused ENI): YELLOW — deletes ENI, recoverable if not in use"
  - "create-subnet (create larger subnet): YELLOW — creates new subnet, recoverable by deleting"
  - "associate-vpc-cidr-block (add secondary CIDR): YELLOW — adds CIDR, recoverable by disassociating"
```

## Escalation Conditions

- "IP exhaustion preventing new resource launches — CRITICAL impact"
- "Fix requires creating new subnets in production VPC"
- "Fix requires adding secondary CIDR to production VPC"
- "Lambda or ELB IP consumption causing exhaustion — managed service impact"

## Data Sensitivity

- HIGH: ENI details (expose resource placement and IP assignments)
- HIGH: subnet CIDR ranges (expose network architecture)
- MEDIUM: VPC CIDR blocks, available IP counts, resource ownership

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete ENIs that are attached to running resources"

## Phase 3 — Rollback

- If unused ENIs were deleted: they cannot be restored — ensure they were truly unused before deletion
- If a new subnet was created: delete it with `aws ec2 delete-subnet --subnet-id <subnet-id>` (must be empty)
- If a secondary CIDR was added: disassociate it with `aws ec2 disassociate-vpc-cidr-block --association-id <assoc-id>` (must delete subnets first)
- Document ENI ownership, subnet configurations, and VPC CIDRs before changes

## Output Format

```yaml
root_cause: "ip_exhaustion — <subnet and available count>"
evidence:
  - type: subnet
    content: "<subnet ID, CIDR, available IPs, top consumers>"
severity: CRITICAL
mitigation:
  immediate: "Clean up unused ENIs or move workloads to larger subnet"
  long_term: "Right-size subnets, add secondary CIDRs, use prefix delegation"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
