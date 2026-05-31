---
title: "B2 — Pool CIDR Provisioning"
description: "Diagnose IPAM pool CIDR provisioning failures"
status: active
severity: HIGH
triggers: ["CIDR provisioning failed", "pending-provision", "provision error", "pool has no CIDRs"]
owner: devops-agent
objective: "Resolve CIDR provisioning issues for IPAM pools"
context: "CIDRs must be provisioned to pools before allocations can be made. Provisioning is asynchronous."
---

## Phase 1 — Triage

MUST:
- Check pool CIDRs: `aws ec2 get-ipam-pool-cidrs --ipam-pool-id <pool-id> --query 'IpamPoolCidrs[*].{Cidr:Cidr,State:State,FailureReason:FailureReason}'`
- Verify the CIDR is valid for the pool's address family
- Check if the CIDR overlaps with existing pool CIDRs

SHOULD:
- For child pools, verify parent pool has the CIDR provisioned
- Check CIDR state transitions (pending-provision → provisioned or failed-provision)

MAY:
- Review CloudTrail for ProvisionIpamPoolCidr events
- Check if BYOIP CIDRs require additional verification

## Phase 2 — Remediate

MUST:
- Fix CIDR format and ensure it matches the pool's address family (IPv4/IPv6)
- For child pools, ensure the CIDR falls within the parent pool's provisioned range
- Wait for provisioning to complete before attempting allocations

SHOULD:
- Provision CIDRs that align with the pool hierarchy plan
- Use appropriate CIDR sizes for the expected allocation patterns

MAY:
- Provision multiple CIDRs to a pool for capacity
- Set up monitoring for provisioning state changes

## Common Issues

- symptoms: "ProvisionIpamPoolCidr returns InvalidParameterValue"
  diagnosis: "CIDR format is invalid or does not match the pool's address family."
  resolution: "Verify CIDR notation (e.g., 10.0.0.0/8 for IPv4). Ensure address family matches pool configuration."

- symptoms: "CIDR state stuck in pending-provision"
  diagnosis: "For BYOIP CIDRs, RIR verification may be pending. For regular CIDRs, there may be an overlap."
  resolution: "Check FailureReason field. For BYOIP, verify RIR authorization. For overlaps, use a different CIDR."

## Safety Ratings

```
safety_ratings:
  - "get-ipam-pool-cidrs: GREEN — read-only CIDR listing"
  - "provision-ipam-pool-cidr: YELLOW — adds CIDR to pool"
  - "deprovision-ipam-pool-cidr: RED — removes CIDR from pool, may affect allocations"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- CIDR provisioning fails repeatedly with no clear error message
- BYOIP CIDR verification is stuck at the RIR level
- Parent pool capacity is exhausted and cannot accommodate new child pool CIDRs
- CIDR overlap detected with another organization's IPAM

## Data Sensitivity

- `aws ec2 get-ipam-pool-cidrs`: MEDIUM — reveals IP address ranges
- CIDR ranges: MEDIUM — reveals network architecture
- BYOIP verification details: MEDIUM — contains RIR authorization information
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER deprovision CIDRs that have active allocations
- NEVER provision overlapping CIDRs to the same pool
- NEVER provision public IP CIDRs to private scope pools
- NEVER skip waiting for provisioning to complete before allocating

## Phase 3 — Rollback

1. If a CIDR was provisioned incorrectly, deprovision it (only if no allocations): `aws ec2 deprovision-ipam-pool-cidr --ipam-pool-id <pool-id> --cidr <cidr>`
2. Wait for deprovision to complete (state: deprovisioned)
3. Provision the correct CIDR: `aws ec2 provision-ipam-pool-cidr --ipam-pool-id <pool-id> --cidr <correct-cidr>`
4. Verify the new CIDR reaches provisioned state
5. Document the CIDR change and update network documentation

## Output Format

```yaml
root_cause: "cidr_provisioning — <specific_cause>"
evidence:
  - type: provisioning_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix CIDR or wait for provisioning"
  long_term: "Automate CIDR provisioning via IaC"
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
