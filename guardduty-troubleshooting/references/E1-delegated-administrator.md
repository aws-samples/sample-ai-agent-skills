---
title: "E1 — Delegated Administrator"
description: "Diagnose GuardDuty delegated administrator setup and management issues"
status: active
severity: HIGH
triggers:
  - "delegated administrator"
  - "delegated admin guardduty"
  - "admin account"
  - "organization guardduty"
  - "admin setup"
owner: devops-agent
objective: "Identify and resolve delegated administrator configuration issues for GuardDuty organization management"
context: "The delegated administrator manages GuardDuty for all member accounts in the organization. Setup requires Organizations access and proper registration. Only one delegated admin per organization per region. The delegated admin controls feature enablement, finding management, and member account settings."
---

## Phase 1 — Triage

MUST:
- Check current admin account: `aws guardduty list-organization-admin-accounts --max-results 5`
- Verify Organizations access: `aws organizations describe-organization`
- Check if GuardDuty trusted access is enabled: `aws organizations list-aws-service-access-for-organization`
- Verify the detector in the admin account: `aws guardduty list-detectors`
- Check organization configuration: `aws guardduty describe-organization-configuration --detector-id <detector-id>`

SHOULD:
- Verify the admin account is a member of the organization: `aws organizations describe-account --account-id <account-id>`
- Check CloudTrail for admin designation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=EnableOrganizationAdminAccount --max-results 10`

MAY:
- Verify the admin account has GuardDuty properly configured
- Check for SCPs blocking GuardDuty operations

## Phase 2 — Remediate

MUST:
- Designate the delegated admin from the management account: `aws guardduty enable-organization-admin-account --admin-account-id <account-id>`
- Enable trusted access for GuardDuty: `aws organizations enable-aws-service-access --service-principal guardduty.amazonaws.com`
- Verify the designation was successful

SHOULD:
- Configure auto-enable for new member accounts
- Enable all required features in the organization configuration

MAY:
- Document the delegated admin setup
- Set up monitoring for admin account changes

## Common Issues

- symptoms: "Cannot designate delegated admin — AccessDeniedException"
  diagnosis: "The caller is not the management account or lacks Organizations permissions."
  resolution: "Run the command from the management account with organizations:EnableAWSServiceAccess and guardduty:EnableOrganizationAdminAccount permissions."

- symptoms: "Delegated admin cannot manage member accounts"
  diagnosis: "Trusted access is not enabled or the admin designation is incomplete."
  resolution: "Enable trusted access for guardduty.amazonaws.com and verify the admin account is properly designated."

- symptoms: "New member accounts not automatically getting GuardDuty enabled"
  diagnosis: "Auto-enable is not configured in the organization configuration."
  resolution: "Update organization configuration to auto-enable GuardDuty and desired features for new accounts."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Designate delegated admin | YELLOW | Organization change; reversible |
| Enable trusted access | YELLOW | Organization change; reversible |
| Configure auto-enable | YELLOW | Organization change; reversible |

## Escalation Conditions

- Delegated admin designation changes
- Organization-wide GuardDuty configuration changes
- Detector configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-organization-admin-accounts` | MEDIUM | Admin account IDs |
| `describe-organization` | MEDIUM | Organization details |
| `describe-organization-configuration` | MEDIUM | Org-wide settings |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Admin designation | Remove via `disable-organization-admin-account` |
| Trusted access | Disable via `disable-aws-service-access` |
| Auto-enable config | Revert via `update-organization-configuration` |

## Output Format

```yaml
root_cause: "delegated_administrator — <specific_cause>"
evidence:
  - type: admin_accounts
    content: "<organization admin accounts>"
  - type: org_config
    content: "<organization configuration>"
  - type: trusted_access
    content: "<service access status>"
severity: HIGH
mitigation:
  immediate: "Designate the delegated admin and enable trusted access"
  long_term: "Configure auto-enable and monitor admin account status"
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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
