---
title: "E1 — Cross-Account Sharing"
description: "Diagnose cross-account license sharing issues"
status: active
severity: HIGH
triggers: ["cross-account license", "share license", "RAM sharing", "license sharing failed"]
owner: devops-agent
objective: "Resolve cross-account license sharing issues"
context: "Cross-account sharing uses AWS RAM and License Manager grants. Organization sharing must be enabled for org-wide distribution."
---

## Phase 1 — Triage

MUST:
- Check RAM sharing status: `aws ram get-resource-share-associations --association-type PRINCIPAL --query 'resourceShareAssociations[*].{Id:resourceShareArn,Status:status}'`
- Verify organization sharing is enabled: `aws ram enable-sharing-with-aws-organization` (check if already enabled)
- List distributed grants: `aws license-manager list-distributed-grants`

SHOULD:
- Check License Manager organization settings: `aws license-manager get-service-settings`
- Verify the target account can receive shares: `aws ram list-principals --resource-owner SELF`

MAY:
- Review RAM resource shares for license configurations
- Check for SCP restrictions on RAM or License Manager

## Phase 2 — Remediate

MUST:
- Enable RAM organization sharing: `aws ram enable-sharing-with-aws-organization`
- Configure License Manager service settings for cross-account: `aws license-manager update-service-settings --enable-cross-account-discovery`
- Create grants for target accounts with appropriate operations

SHOULD:
- Create RAM resource shares for license configurations: `aws ram create-resource-share --name <name> --resource-arns <license-config-arn> --principals <account-id>`
- Verify sharing works by checking received resources in target account

MAY:
- Automate sharing for new organization members
- Set up monitoring for cross-account license consumption

## Safety Ratings

```
safety_ratings:
  - "ram get-resource-share-associations: GREEN — read-only listing"
  - "ram enable-sharing-with-aws-organization: YELLOW — enables org-wide sharing"
  - "ram create-resource-share: YELLOW — shares resources across accounts"
  - "license-manager update-service-settings: YELLOW — modifies service configuration"
```

## Escalation Conditions

- RAM sharing fails with organization trust errors
- Cross-account discovery reveals unexpected license consumption
- Sharing configuration affects other RAM-shared resources
- Management account access required for organization settings

## Data Sensitivity

- RAM resource shares: MEDIUM — reveals cross-account relationships
- License consumption across accounts: MEDIUM — reveals org-wide usage
- Organization settings: MEDIUM — reveals governance configuration
- Account identifiers: LOW — account numbers

## Prohibited Actions

- NEVER share licenses with accounts outside the organization without approval
- NEVER enable organization sharing without understanding impact on all RAM shares
- NEVER disable cross-account discovery during compliance reviews
- NEVER modify RAM shares that include non-license resources without coordination

## Phase 3 — Rollback

1. Delete RAM resource shares if created in error: `aws ram delete-resource-share --resource-share-arn <arn>`
2. Revoke grants if shared incorrectly: `aws license-manager delete-grant --grant-arn <arn>`
3. Disable cross-account discovery if enabled incorrectly
4. Verify license consumption is correct after sharing changes
5. Notify affected accounts of sharing changes

## Common Issues

- symptoms: "RAM resource share shows FAILED status"
  diagnosis: "Organization sharing not enabled or target account not in the organization."
  resolution: "Enable RAM organization sharing. Verify target account is an active organization member."

## Output Format

```yaml
root_cause: "cross_account_sharing — <specific_cause>"
evidence:
  - type: sharing_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix cross-account sharing configuration"
  long_term: "Automate license sharing for organization members"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
