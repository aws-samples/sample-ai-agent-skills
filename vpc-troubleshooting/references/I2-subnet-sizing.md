---
title: "I2 — Subnet Sizing Issues"
description: "Diagnose problems caused by incorrectly sized subnets"
status: active
severity: MEDIUM
triggers:
  - "Subnet too small"
  - "Need more IPs"
  - "Subnet planning"
owner: devops-agent
objective: "Identify subnet sizing issues and plan remediation"
context: "Subnet CIDRs cannot be resized after creation. 5 IPs reserved per subnet. Common mistake: /28 (11 usable IPs) for workloads that grow. Plan for growth. Secondary CIDRs can extend the VPC but require new subnets."
---

## Phase 1 — Triage

MUST:
- Check current subnet CIDR and available IPs
- Calculate total usable IPs (2^(32-prefix) - 5)
- Assess current and projected IP consumption
- Check if VPC has room for additional subnets

SHOULD:
- Review VPC CIDR and secondary CIDRs
- Check if other subnets in the same AZ have available capacity
- Evaluate if workloads can be redistributed across subnets

MAY:
- Plan a migration to larger subnets
- Evaluate adding a secondary CIDR to the VPC

## Common Issues

- symptoms: "/28 subnet running out of IPs with only 11 usable"
  diagnosis: "Subnet too small for the workload. Cannot resize existing subnet."
  resolution: "Create a new larger subnet in the same AZ, migrate resources, delete old subnet."

- symptoms: "VPC CIDR fully allocated, can't create new subnets"
  diagnosis: "Primary VPC CIDR exhausted."
  resolution: "Add a secondary CIDR block to the VPC, then create new subnets from it."

## Safety Ratings

```
safety_ratings:
  - "describe-subnets: GREEN — read-only subnet CIDR and IP count check"
  - "describe-vpcs: GREEN — read-only VPC CIDR inspection"
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "create-subnet (create larger replacement subnet): YELLOW — creates new subnet, recoverable by deleting"
  - "associate-vpc-cidr-block (add secondary CIDR): YELLOW — adds CIDR, recoverable by disassociating"
  - "delete-subnet: RED — deletes subnet, all resources must be moved first"
```

## Escalation Conditions

- "Fix requires creating new subnets in production VPC"
- "Fix requires migrating resources between subnets — service disruption possible"
- "Fix requires adding secondary CIDR to production VPC"
- "VPC CIDR fully allocated — no room for new subnets"

## Data Sensitivity

- HIGH: subnet CIDR ranges (expose network architecture and IP planning)
- HIGH: VPC CIDR blocks (expose overall network design)
- MEDIUM: route table associations, available IP counts

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete a subnet with active resources"

## Phase 3 — Rollback

- If a new subnet was created: delete it with `aws ec2 delete-subnet --subnet-id <subnet-id>` (must be empty)
- If a secondary CIDR was added: disassociate it with `aws ec2 disassociate-vpc-cidr-block --association-id <assoc-id>` (must delete subnets in that CIDR first)
- If resources were migrated to new subnet: migrate them back to original subnet (if it still exists)
- Document subnet configurations, route table associations, and resource placements before changes

## Output Format

```yaml
root_cause: "subnet_sizing — <detail>"
evidence:
  - type: subnet
    content: "<subnet CIDR, usable IPs, consumption>"
severity: MEDIUM
mitigation:
  immediate: "Create larger subnet and migrate resources"
  long_term: "Plan subnet sizes with growth in mind, use /24 minimum for workload subnets"
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
