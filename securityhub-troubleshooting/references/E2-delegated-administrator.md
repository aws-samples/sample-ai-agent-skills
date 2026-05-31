---
title: "E2 — Delegated Administrator"
description: "Diagnose Security Hub delegated administrator setup and management issues"
status: active
severity: HIGH
triggers:
  - "delegated administrator"
  - "delegated admin security hub"
  - "admin account setup"
  - "organization admin"
  - "admin designation"
owner: devops-agent
objective: "Identify and resolve delegated administrator configuration issues for Security Hub"
context: "The delegated administrator manages Security Hub for all member accounts in the organization. Setup requires Organizations access and proper registration. The delegated admin can enable standards, manage findings, and configure settings for all members."
---

## Phase 1 — Triage

MUST:
- Check current admin: `aws securityhub list-organization-admin-accounts`
- Verify Organizations access: `aws organizations describe-organization`
- Check trusted access: `aws organizations list-aws-service-access-for-organization`
- Verify Security Hub is enabled in the admin account: `aws securityhub describe-hub`
- Check organization configuration: `aws securityhub describe-organization-configuration`

SHOULD:
- Verify the admin account is a member of the organization
- Check CloudTrail for admin designation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=EnableOrganizationAdminAccount --max-results 10`

MAY:
- Check for SCPs blocking Security Hub operations
- Verify the admin account has proper IAM permissions

## Phase 2 — Remediate

MUST:
- Designate the delegated admin from the management account: `aws securityhub enable-organization-admin-account --admin-account-id <account-id>`
- Enable trusted access: `aws organizations enable-aws-service-access --service-principal securityhub.amazonaws.com`
- Verify the designation was successful

SHOULD:
- Configure auto-enable for new member accounts
- Enable desired security standards for the organization

MAY:
- Document the delegated admin setup
- Set up monitoring for admin account changes

## Common Issues

- symptoms: "Cannot designate delegated admin — AccessDeniedException"
  diagnosis: "The caller is not the management account or lacks Organizations permissions."
  resolution: "Run from the management account with organizations:EnableAWSServiceAccess and securityhub:EnableOrganizationAdminAccount."

- symptoms: "Delegated admin cannot manage member accounts"
  diagnosis: "Trusted access is not enabled or the designation is incomplete."
  resolution: "Enable trusted access for securityhub.amazonaws.com and verify the admin designation."

- symptoms: "Delegated admin designated but organization configuration not working"
  diagnosis: "The admin needs to configure organization settings after designation."
  resolution: "Use update-organization-configuration to set auto-enable and standard settings."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Designate delegated admin | YELLOW | Organization change; reversible |
| Enable trusted access | YELLOW | Organization change; reversible |
| Configure organization settings | YELLOW | Organization change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Delegated admin designation changes
- Organization-wide Security Hub configuration

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-organization-admin-accounts` | MEDIUM | Admin account IDs |
| `describe-organization` | MEDIUM | Organization details |
| `describe-organization-configuration` | MEDIUM | Org-wide settings |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Admin designation | Remove via `disable-organization-admin-account` |
| Trusted access | Disable via `disable-aws-service-access` |
| Organization config | Revert via `update-organization-configuration` |

## Output Format

```yaml
root_cause: "delegated_administrator — <specific_cause>"
evidence:
  - type: admin_accounts
    content: "<organization admin accounts>"
  - type: trusted_access
    content: "<service access status>"
  - type: org_config
    content: "<organization configuration>"
severity: HIGH
mitigation:
  immediate: "Designate the admin and enable trusted access"
  long_term: "Configure organization settings and monitor admin status"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
