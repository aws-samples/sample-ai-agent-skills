---
title: "C2 — CIDR Overlap Detection"
description: "Diagnose IPAM CIDR overlap detection and resolution"
status: active
severity: HIGH
triggers: ["CIDR overlap", "overlapping CIDRs", "duplicate CIDR", "conflict detection"]
owner: devops-agent
objective: "Detect and resolve CIDR overlaps across the organization"
context: "IPAM detects overlaps through resource discovery. Overlaps from manual CIDR assignments are detected but not prevented."
---

## Phase 1 — Triage

MUST:
- Check resource CIDRs for overlaps: `aws ec2 get-ipam-resource-cidrs --ipam-scope-id <scope-id> --query 'IpamResourceCidrs[*].{ResourceId:ResourceId,ResourceCidr:ResourceCidr,OverlapStatus:OverlapStatus,VpcId:VpcId}'`
- Identify which resources have overlapping CIDRs
- Verify resource discovery is running and up to date

SHOULD:
- Check resource discovery status: `aws ec2 describe-ipam-resource-discoveries --query 'IpamResourceDiscoveries[*].{DiscoveryId:IpamResourceDiscoveryId,State:State}'`
- Map overlapping CIDRs to accounts and regions

MAY:
- Review VPC peering and Transit Gateway configurations for routing conflicts
- Check if overlapping CIDRs are in isolated environments (no connectivity)

## Phase 2 — Remediate

MUST:
- Identify all resources using overlapping CIDRs
- Determine if overlapping CIDRs cause routing conflicts
- Plan CIDR migration for conflicting resources

SHOULD:
- Migrate manually assigned CIDRs to IPAM-managed allocations
- Implement IPAM allocation policies to prevent future overlaps

MAY:
- Use VPC CIDR association to add non-overlapping CIDRs alongside existing ones
- Implement SCP to require IPAM allocation for VPC creation

## Common Issues

- symptoms: "IPAM shows overlapping CIDRs between two VPCs in different accounts"
  diagnosis: "Both VPCs were created with manually specified CIDRs that overlap."
  resolution: "If VPCs need connectivity, migrate one to a non-overlapping CIDR. If isolated, document the overlap as accepted."

- symptoms: "Overlap detection shows no overlaps but VPC peering fails"
  diagnosis: "Resource discovery may not have completed its latest scan."
  resolution: "Wait for resource discovery to complete. Check discovery status and last scan time."

## Safety Ratings

```
safety_ratings:
  - "get-ipam-resource-cidrs: GREEN — read-only resource CIDR listing"
  - "describe-ipam-resource-discoveries: GREEN — read-only discovery status"
  - "VPC CIDR migration: RED — requires VPC recreation or secondary CIDR"
  - "SCP implementation: RED — affects organization-wide VPC creation"
```

## Escalation Conditions

- Overlapping CIDRs prevent required VPC peering or Transit Gateway connectivity
- CIDR migration requires downtime for production workloads
- Overlaps exist across multiple accounts requiring coordinated remediation
- SCP implementation needed to enforce IPAM allocation

## Data Sensitivity

- `aws ec2 get-ipam-resource-cidrs`: MEDIUM — reveals IP allocations across accounts
- Overlap mapping: MEDIUM — reveals network architecture and conflicts
- VPC-to-account mapping: MEDIUM — reveals organizational structure
- Resource discovery data: MEDIUM — reveals cross-account resource inventory

## Prohibited Actions

- NEVER ignore CIDR overlaps in environments that require connectivity
- NEVER delete VPCs to resolve overlaps without migrating workloads
- NEVER modify VPC CIDRs on production VPCs without a maintenance window
- NEVER assume overlaps are harmless without verifying connectivity requirements

## Phase 3 — Rollback

1. If CIDR changes were made to resolve overlaps, verify connectivity is maintained
2. If a secondary CIDR was added, it can be disassociated if not in use: `aws ec2 disassociate-vpc-cidr-block --association-id <assoc-id>`
3. If SCP was implemented, revert if it blocks legitimate VPC creation
4. Verify resource discovery reflects the current state after changes
5. Document the overlap resolution and update network documentation

## Output Format

```yaml
root_cause: "cidr_overlap — <specific_cause>"
evidence:
  - type: overlap_details
    content: "<overlapping CIDRs and resources>"
severity: HIGH
mitigation:
  immediate: "Resolve routing conflicts"
  long_term: "Enforce IPAM allocation for all VPCs"
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
