---
title: "B1 — Pool Creation Errors"
description: "Diagnose IPAM pool creation failures"
status: active
severity: HIGH
triggers: ["pool creation failed", "cannot create pool", "pool error", "invalid pool configuration"]
owner: devops-agent
objective: "Resolve IPAM pool creation and configuration issues"
context: "Pools are the core allocation unit in IPAM. They belong to scopes and have locale, address family, and allocation rules."
---

## Phase 1 — Triage

MUST:
- List existing pools: `aws ec2 describe-ipam-pools --query 'IpamPools[*].{PoolId:IpamPoolId,State:State,AddressFamily:AddressFamily,Locale:Locale,SourcePoolId:SourceIpamPoolId}'`
- Verify the target scope exists: `aws ec2 describe-ipam-scopes --query 'IpamScopes[*].{ScopeId:IpamScopeId,Type:IpamScopeType}'`
- Check IAM permissions for ec2:CreateIpamPool

SHOULD:
- Verify locale is a valid operating region of the IPAM
- Check if a parent pool exists for nested pool creation
- Verify allocation rules (min/max netmask length) are valid

MAY:
- Review CloudTrail for CreateIpamPool errors
- Check service quotas for IPAM pools

## Phase 2 — Remediate

MUST:
- Fix pool configuration (locale, address family, scope ID)
- Ensure locale matches an IPAM operating region
- Set appropriate allocation rules (netmask length constraints)

SHOULD:
- Create pool hierarchy: top-level pool → regional pools → environment pools
- Tag pools with region, environment, and purpose

MAY:
- Implement auto-import for existing VPC CIDRs
- Configure pool sharing via RAM for cross-account allocation

## Common Issues

- symptoms: "CreateIpamPool returns InvalidParameterValue for locale"
  diagnosis: "Specified locale is not in the IPAM's operating regions."
  resolution: "Add the region to IPAM operating regions: aws ec2 modify-ipam --ipam-id <id> --add-operating-regions RegionName=<region>"

- symptoms: "CreateIpamPool fails for child pool with insufficient parent capacity"
  diagnosis: "Parent pool does not have enough provisioned CIDR space for the child pool."
  resolution: "Provision additional CIDRs to the parent pool before creating child pools."

## Safety Ratings

```
safety_ratings:
  - "describe-ipam-pools: GREEN — read-only pool listing"
  - "describe-ipam-scopes: GREEN — read-only scope listing"
  - "CreateIpamPool: YELLOW — creates pool resource"
  - "modify-ipam: YELLOW — changes IPAM operating regions"
```

## Escalation Conditions

- Pool creation fails with internal errors after fixing configuration
- Pool hierarchy design requires architectural review
- Cross-account pool sharing requires RAM and organization configuration
- Pool creation blocked by SCP or permission boundary

## Data Sensitivity

- `aws ec2 describe-ipam-pools`: LOW — returns pool metadata
- `aws ec2 describe-ipam-scopes`: LOW — returns scope metadata
- Pool allocation rules: LOW — contains netmask constraints
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER create pools without a clear hierarchy plan
- NEVER set allocation rules that are too restrictive for the use case
- NEVER create pools in regions not included in IPAM operating regions
- NEVER skip tagging pools for cost and ownership tracking

## Phase 3 — Rollback

1. If a pool was created with incorrect configuration, delete it (only if no allocations): `aws ec2 delete-ipam-pool --ipam-pool-id <pool-id>`
2. If IPAM operating regions were modified, revert: `aws ec2 modify-ipam --ipam-id <id> --remove-operating-regions RegionName=<region>`
3. Verify parent pool capacity is not affected by child pool deletion
4. Update IaC templates to reflect the correct pool configuration
5. Document the pool hierarchy and allocation rules

## Output Format

```yaml
root_cause: "pool_creation — <specific_cause>"
evidence:
  - type: pool_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix pool configuration"
  long_term: "Implement pool hierarchy via IaC"
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
