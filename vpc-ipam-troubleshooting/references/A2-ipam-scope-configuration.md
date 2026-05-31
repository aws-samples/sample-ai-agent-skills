---
title: "A2 — IPAM Scope Configuration"
description: "Diagnose IPAM scope configuration issues"
status: active
severity: MEDIUM
triggers: ["scope error", "wrong scope", "public scope", "private scope", "custom scope"]
owner: devops-agent
objective: "Resolve IPAM scope configuration and management issues"
context: "IPAM has two default scopes (public, private) and supports custom scopes for additional isolation."
---

## Phase 1 — Triage

MUST:
- List scopes: `aws ec2 describe-ipam-scopes --query 'IpamScopes[*].{ScopeId:IpamScopeId,Type:IpamScopeType,IsDefault:IsDefault,PoolCount:PoolCount}'`
- Verify pools are in the correct scope (public vs private)
- Check if custom scopes are needed for the use case

SHOULD:
- Verify scope-to-pool mapping matches the IP address family (public/private)
- Check if pools in the wrong scope are causing allocation issues

MAY:
- Review scope tags for organizational context
- Check if custom scopes are used for environment isolation

## Phase 2 — Remediate

MUST:
- Move pools to the correct scope if misplaced (requires recreating the pool)
- Ensure public IP pools are in the public scope and private IP pools in the private scope

SHOULD:
- Create custom scopes for environment isolation (dev, staging, prod)
- Document scope-to-pool mapping

MAY:
- Use tags to identify scope purpose
- Implement naming conventions for scopes

## Common Issues

- symptoms: "Pool creation fails with 'invalid scope' error"
  diagnosis: "Attempting to create a pool with the wrong address family for the scope."
  resolution: "Use the private scope for RFC 1918 CIDRs and the public scope for public IP CIDRs."

- symptoms: "Cannot find pools in the expected scope"
  diagnosis: "Pools were created in the wrong scope (public vs private)."
  resolution: "Recreate pools in the correct scope. Pools cannot be moved between scopes."

## Safety Ratings

```
safety_ratings:
  - "describe-ipam-scopes: GREEN — read-only scope listing"
  - "create-ipam-scope: YELLOW — creates custom scope"
  - "delete-ipam-scope: YELLOW — removes custom scope (default scopes cannot be deleted)"
  - "Pool recreation: RED — requires deallocating and reprovisioning CIDRs"
```

## Escalation Conditions

- Pools in the wrong scope require migration affecting production VPCs
- Custom scope requirements conflict with existing IPAM structure
- Scope configuration changes needed across multiple accounts
- Default scope limitations require architectural changes

## Data Sensitivity

- `aws ec2 describe-ipam-scopes`: LOW — returns scope metadata
- Scope configuration: LOW — contains scope type and pool count
- Pool-to-scope mapping: LOW — reveals IP management structure

## Prohibited Actions

- NEVER delete default scopes (they cannot be deleted, but do not attempt)
- NEVER create pools in the public scope for private IP ranges
- NEVER modify scope configuration without understanding the pool hierarchy
- NEVER delete custom scopes with active pools

## Phase 3 — Rollback

1. If a custom scope was created in error, delete it (only if no pools exist): `aws ec2 delete-ipam-scope --ipam-scope-id <scope-id>`
2. If pools were created in the wrong scope, plan migration to the correct scope
3. Verify existing allocations are not affected by scope changes
4. Document the scope configuration and purpose
5. Update IaC templates to reflect the correct scope configuration

## Output Format

```yaml
root_cause: "scope_configuration — <specific_cause>"
evidence:
  - type: scope_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix scope-pool mapping"
  long_term: "Document and standardize scope usage"
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
