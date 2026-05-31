---
title: "D1 — Organization Integration"
description: "Diagnose IPAM organization integration and cross-account issues"
status: active
severity: HIGH
triggers: ["organization integration", "cross-account IPAM", "RAM sharing", "resource discovery", "member account"]
owner: devops-agent
objective: "Resolve IPAM organization integration and cross-account visibility issues"
context: "IPAM uses resource discovery and RAM sharing for cross-account IP management."
---

## Phase 1 — Triage

MUST:
- Check resource discoveries: `aws ec2 describe-ipam-resource-discoveries --query 'IpamResourceDiscoveries[*].{DiscoveryId:IpamResourceDiscoveryId,State:State,IsDefault:IsDefault,OperatingRegions:OperatingRegions}'`
- Verify RAM sharing: `aws ram get-resource-shares --resource-owner SELF --query 'resourceShares[*].{Name:name,Status:status}'`
- Check if IPAM delegated admin is configured: `aws organizations list-delegated-administrators --service-principal ipam.amazonaws.com`

SHOULD:
- Verify resource discovery association with IPAM: `aws ec2 describe-ipam-resource-discovery-associations --query 'IpamResourceDiscoveryAssociations[*].{AssociationId:IpamResourceDiscoveryAssociationId,State:State}'`
- Check if member accounts can see shared pools

MAY:
- Review RAM resource share permissions
- Check Organizations trusted access for IPAM

## Phase 2 — Remediate

MUST:
- Enable trusted access for IPAM in Organizations: `aws organizations enable-aws-service-access --service-principal ipam.amazonaws.com`
- Create and share resource discovery via RAM
- Associate resource discovery with IPAM

SHOULD:
- Register IPAM delegated administrator for non-management account operation
- Share pools via RAM for cross-account allocation

MAY:
- Configure resource discovery operating regions to match IPAM
- Set up automated resource discovery refresh

## Common Issues

- symptoms: "IPAM does not show resources from member accounts"
  diagnosis: "Resource discovery is not configured or not shared via RAM."
  resolution: "Create resource discovery, associate with IPAM, share via RAM to the organization."

- symptoms: "Member account cannot allocate from shared IPAM pool"
  diagnosis: "Pool is not shared via RAM or member account lacks ec2:AllocateIpamPoolCidr permission."
  resolution: "Share the pool via RAM. Ensure member account IAM role has ec2:AllocateIpamPoolCidr."

## Safety Ratings

```
safety_ratings:
  - "describe-ipam-resource-discoveries: GREEN — read-only discovery listing"
  - "ram get-resource-shares: GREEN — read-only RAM share listing"
  - "organizations enable-aws-service-access: RED — enables org-level service access"
  - "ram create-resource-share: YELLOW — shares resources across accounts"
```

## Escalation Conditions

- Organization trusted access cannot be enabled (requires management account)
- RAM sharing fails due to organization SCP restrictions
- Resource discovery does not detect resources in specific member accounts
- Delegated admin registration requires management account approval

## Data Sensitivity

- `aws ec2 describe-ipam-resource-discoveries`: LOW — returns discovery metadata
- `aws ram get-resource-shares`: MEDIUM — reveals cross-account sharing configuration
- `aws organizations list-delegated-administrators`: MEDIUM — reveals admin accounts
- Resource discovery data: MEDIUM — reveals cross-account resource inventory

## Prohibited Actions

- NEVER enable organization trusted access without management account owner approval
- NEVER share IPAM pools with accounts outside the organization
- NEVER register delegated admin without understanding the security implications
- NEVER disable resource discovery without understanding the impact on compliance monitoring

## Phase 3 — Rollback

1. If RAM sharing was configured incorrectly, delete the resource share: `aws ram delete-resource-share --resource-share-arn <arn>`
2. If trusted access was enabled in error, disable it: `aws organizations disable-aws-service-access --service-principal ipam.amazonaws.com`
3. If delegated admin was registered incorrectly: `aws organizations deregister-delegated-administrator --account-id <id> --service-principal ipam.amazonaws.com`
4. Verify IPAM functionality is not disrupted after rollback
5. Document the organization integration changes and outcome

## Output Format

```yaml
root_cause: "organization_integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix RAM sharing or resource discovery"
  long_term: "Automate organization integration via IaC"
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
