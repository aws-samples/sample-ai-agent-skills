---
title: "F1 — RAM Sharing Errors"
description: "Diagnose AWS RAM sharing errors for resolver rules and DNS Firewall rule groups"
status: active
severity: MEDIUM
triggers: ["RAM sharing failed", "cannot share rule", "cross-account sharing error"]
owner: devops-agent
objective: "Resolve RAM sharing issues for Route 53 Resolver resources"
context: "Resolver rules and DNS Firewall rule groups can be shared across accounts via AWS RAM. Sharing requires Organization membership by default. External sharing must be explicitly enabled."
---

## Phase 1 — Triage

MUST:
- List RAM resource shares: `aws ram get-resource-shares --resource-owner SELF --query 'resourceShares[*].{Name:name,Status:status,AllowExternal:allowExternalPrincipals}'`
- Check shared resources: `aws ram list-resources --resource-owner SELF --resource-type route53resolver:ResolverRule`
- Verify Organization membership: `aws organizations describe-organization`

SHOULD:
- Check if external sharing is enabled in RAM: `aws ram get-resource-share-invitations`
- Verify the target account has accepted the share invitation

MAY:
- Review CloudTrail for RAM sharing errors
- Check if SCPs restrict RAM sharing operations

## Phase 2 — Remediate

MUST:
- Ensure both accounts are in the same Organization (or enable external sharing)
- Create RAM resource share: `aws ram create-resource-share --name <name> --resource-arns <rule-arn> --principals <account-id>`
- Verify the share status is ACTIVE

SHOULD:
- Accept share invitation in the target account if required
- Verify the shared resource appears in the target account

MAY:
- Enable external sharing if accounts are not in the same Organization
- Use Organization-level sharing for automatic propagation to all member accounts

## Common Issues

- symptoms: "RAM share creation fails with 'principal not in organization'"
  diagnosis: "Target account is not in the same Organization and external sharing is disabled."
  resolution: "Enable external sharing in RAM settings or ensure both accounts are in the same Organization."

- symptoms: "Shared rule not visible in target account"
  diagnosis: "Share invitation not accepted or share status not ACTIVE."
  resolution: "Accept the share invitation in the target account. Verify share status."

## Safety Ratings

```
safety_ratings:
  - "ram get-resource-shares: GREEN — read-only share listing"
  - "ram list-resources: GREEN — read-only resource listing"
  - "ram create-resource-share: YELLOW — shares resources with other accounts"
  - "ram enable-sharing-with-aws-organization: YELLOW — enables org-wide sharing"
```

## Escalation Conditions

- RAM sharing fails due to Organization-level restrictions
- SCPs prevent RAM operations in the sharing or target account
- Shared resources need to be unshared but are in active use
- Cross-region sharing is required but not supported

## Data Sensitivity

- `aws ram get-resource-shares`: MEDIUM — reveals cross-account sharing relationships
- `aws ram list-resources`: MEDIUM — reveals shared resource ARNs
- Organization membership: LOW — confirms organizational structure
- Share invitations: MEDIUM — reveal account IDs and sharing intent

## Prohibited Actions

- NEVER share resolver rules or firewall groups with untrusted accounts
- NEVER enable external sharing without security team approval
- NEVER delete RAM shares that target accounts are actively using
- NEVER share resources containing sensitive DNS configuration outside the organization

## Phase 3 — Rollback

1. If a share was created incorrectly, delete it: `aws ram delete-resource-share --resource-share-arn <arn>`
2. If external sharing was enabled, disable it if not needed
3. If resources were shared with wrong accounts, update the share principals
4. Verify target accounts are not affected by share removal
5. Document sharing changes for the security team

## Output Format

```yaml
root_cause: "ram_sharing — <specific_cause>"
evidence:
  - type: sharing_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix RAM sharing configuration"
  long_term: "Automate cross-account sharing via Organizations"
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
  - "NEVER suggest disabling DNSSEC without understanding impact"
  - "NEVER suggest wildcard DNS records without review"
  - "NEVER modify hosted zones without backup"
