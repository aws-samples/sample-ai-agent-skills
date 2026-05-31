---
title: "C1 — Allocation Failures"
description: "Diagnose IPAM pool allocation failures"
status: active
severity: HIGH
triggers: ["allocation failed", "cannot allocate CIDR", "insufficient pool capacity", "netmask error"]
owner: devops-agent
objective: "Resolve CIDR allocation failures from IPAM pools"
context: "Allocations assign CIDRs from pools to VPCs or other resources. Failures indicate capacity, locale, or rule issues."
---

## Phase 1 — Triage

MUST:
- Check pool allocations: `aws ec2 get-ipam-pool-allocations --ipam-pool-id <pool-id> --query 'IpamPoolAllocations[*].{Cidr:Cidr,ResourceId:ResourceId,ResourceType:ResourceType}'`
- Check pool capacity: `aws ec2 get-ipam-pool-cidrs --ipam-pool-id <pool-id> --query 'IpamPoolCidrs[*].{Cidr:Cidr,State:State}'`
- Verify pool locale matches the target resource region

SHOULD:
- Check pool allocation rules (min/max netmask length): `aws ec2 describe-ipam-pools --ipam-pool-ids <pool-id> --query 'IpamPools[0].{AllocationMinNetmaskLength:AllocationMinNetmaskLength,AllocationMaxNetmaskLength:AllocationMaxNetmaskLength,AllocationDefaultNetmaskLength:AllocationDefaultNetmaskLength}'`
- Calculate remaining free space in the pool

MAY:
- Review recent allocation history via CloudTrail
- Check if pool is shared via RAM and allocation is from a member account

## Phase 2 — Remediate

MUST:
- Provision additional CIDRs to the pool if capacity is exhausted
- Fix netmask length to comply with pool allocation rules
- Ensure pool locale matches the target VPC region

SHOULD:
- Release unused allocations to free capacity
- Adjust allocation rules if they are too restrictive

MAY:
- Create additional pools for capacity expansion
- Implement allocation monitoring and alerting

## Common Issues

- symptoms: "CreateVpc with IPAM pool returns InsufficientCidrCapacity"
  diagnosis: "Pool does not have enough free CIDR space for the requested netmask length."
  resolution: "Provision additional CIDRs to the pool or release unused allocations."

- symptoms: "Allocation fails with 'netmask length not within pool rules'"
  diagnosis: "Requested netmask length is outside the pool's min/max allocation rules."
  resolution: "Request a netmask length within the pool's AllocationMinNetmaskLength and AllocationMaxNetmaskLength."

## Safety Ratings

```
safety_ratings:
  - "get-ipam-pool-allocations: GREEN — read-only allocation listing"
  - "get-ipam-pool-cidrs: GREEN — read-only CIDR listing"
  - "describe-ipam-pools: GREEN — read-only pool details"
  - "release-ipam-pool-allocation: RED — releases CIDR allocation"
```

## Escalation Conditions

- Pool capacity exhausted and no additional CIDRs available for provisioning
- Allocation failures affect VPC creation in production
- Cross-account allocation fails due to RAM sharing issues
- Pool hierarchy redesign needed to accommodate growth

## Data Sensitivity

- `aws ec2 get-ipam-pool-allocations`: MEDIUM — reveals IP allocations and resource IDs
- `aws ec2 get-ipam-pool-cidrs`: MEDIUM — reveals IP address ranges
- Pool allocation rules: LOW — contains netmask constraints
- Resource-to-CIDR mapping: MEDIUM — reveals network architecture

## Prohibited Actions

- NEVER release allocations for active VPCs without understanding the impact
- NEVER allocate CIDRs from pools with the wrong locale
- NEVER bypass IPAM allocation by manually specifying CIDRs for VPCs
- NEVER ignore allocation failures — they indicate capacity planning issues

## Phase 3 — Rollback

1. If an allocation was made in error, release it: `aws ec2 release-ipam-pool-allocation --ipam-pool-id <pool-id> --cidr <cidr> --ipam-pool-allocation-id <alloc-id>`
2. If a VPC was created with the wrong CIDR, plan VPC migration
3. If pool rules were changed, revert: `aws ec2 modify-ipam-pool --ipam-pool-id <pool-id> --allocation-min-netmask-length <original> --allocation-max-netmask-length <original>`
4. Verify remaining allocations are not affected
5. Document the allocation failure and capacity status

## Output Format

```yaml
root_cause: "allocation_failure — <specific_cause>"
evidence:
  - type: allocation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix capacity or allocation rules"
  long_term: "Implement capacity monitoring and planning"
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
