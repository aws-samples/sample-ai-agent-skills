---
title: "D2 — Cross-Region Configuration"
description: "Diagnose IPAM cross-region configuration and pool hierarchy issues"
status: active
severity: MEDIUM
triggers: ["cross-region", "wrong region", "locale mismatch", "operating regions", "multi-region"]
owner: devops-agent
objective: "Resolve cross-region IPAM configuration and pool locale issues"
context: "IPAM supports multi-region management through operating regions and pool locale configuration."
---

## Phase 1 — Triage

MUST:
- Check IPAM operating regions: `aws ec2 describe-ipams --query 'Ipams[*].{IpamId:IpamId,OperatingRegions:OperatingRegions}'`
- Verify pool locales match target regions: `aws ec2 describe-ipam-pools --query 'IpamPools[*].{PoolId:IpamPoolId,Locale:Locale,AddressFamily:AddressFamily}'`
- Confirm the target region is in the IPAM operating regions list

SHOULD:
- Map pool hierarchy to understand regional allocation paths
- Verify resource discovery covers all target regions

MAY:
- Check if VPCs in the target region are visible to IPAM
- Review cross-region network architecture requirements

## Phase 2 — Remediate

MUST:
- Add missing regions to IPAM operating regions: `aws ec2 modify-ipam --ipam-id <ipam-id> --add-operating-regions RegionName=<region>`
- Create regional pools with correct locale for each target region
- Ensure pool hierarchy supports the multi-region allocation pattern

SHOULD:
- Create a top-level pool (no locale) with regional child pools
- Provision appropriate CIDRs to each regional pool

MAY:
- Implement automated pool creation for new regions
- Use CloudFormation StackSets for multi-region IPAM pool deployment

## Common Issues

- symptoms: "Cannot allocate CIDR for VPC in eu-west-1 from IPAM pool"
  diagnosis: "Pool locale is set to us-east-1, not eu-west-1."
  resolution: "Create a pool with locale eu-west-1 under the top-level pool. Provision CIDRs to the new pool."

- symptoms: "IPAM does not discover resources in ap-southeast-1"
  diagnosis: "ap-southeast-1 is not in the IPAM operating regions."
  resolution: "Add the region: aws ec2 modify-ipam --ipam-id <id> --add-operating-regions RegionName=ap-southeast-1"

## Safety Ratings

```
safety_ratings:
  - "describe-ipams: GREEN — read-only IPAM details"
  - "describe-ipam-pools: GREEN — read-only pool listing"
  - "modify-ipam operating regions: YELLOW — changes IPAM regional scope"
  - "create-ipam-pool with locale: YELLOW — creates regional pool"
```

## Escalation Conditions

- Adding operating regions requires IPAM modification permissions not available
- Cross-region pool hierarchy redesign needed for organizational changes
- Regional compliance requirements restrict CIDR allocation patterns
- Multi-region deployment requires coordinated pool provisioning

## Data Sensitivity

- `aws ec2 describe-ipams`: LOW — returns IPAM metadata and operating regions
- `aws ec2 describe-ipam-pools`: LOW — returns pool metadata and locales
- Regional pool configuration: LOW — reveals deployment regions
- Cross-region architecture: MEDIUM — reveals multi-region network design

## Prohibited Actions

- NEVER remove operating regions that have active pools or allocations
- NEVER create pools with locale set to a region not in operating regions
- NEVER assume a single pool can serve all regions
- NEVER skip regional pool creation for regions with VPCs

## Phase 3 — Rollback

1. If operating regions were added incorrectly, remove them (only if no pools exist in that region): `aws ec2 modify-ipam --ipam-id <id> --remove-operating-regions RegionName=<region>`
2. If regional pools were created incorrectly, delete them (only if no allocations)
3. Verify existing regional allocations are not affected
4. Update IaC templates to reflect the correct regional configuration
5. Document the cross-region configuration and pool hierarchy

## Output Format

```yaml
root_cause: "cross_region — <specific_cause>"
evidence:
  - type: region_error
    content: "<region and locale details>"
severity: MEDIUM
mitigation:
  immediate: "Fix operating regions or pool locale"
  long_term: "Implement multi-region pool hierarchy via IaC"
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
