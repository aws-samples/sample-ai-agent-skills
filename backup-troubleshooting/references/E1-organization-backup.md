---
title: "E1 — Organization Backup Setup Issues"
description: "Diagnose AWS Organizations backup policy and cross-account backup configuration problems"
status: active
severity: HIGH
triggers:
  - "organization backup"
  - "backup policy"
  - "org backup"
  - "management account backup"
  - "delegated administrator"
owner: devops-agent
objective: "Identify and resolve issues with AWS Organizations backup policies and cross-account backup configuration"
context: "AWS Organizations backup policies allow centralized backup management across member accounts. The management account or a delegated administrator configures backup policies that are inherited by OUs and accounts. Cross-account backup and monitoring must be enabled in the management account's AWS Backup settings."
---

## Phase 1 — Triage

MUST:
- Verify the organization exists: `aws organizations describe-organization`
- Check if backup policies are enabled: `aws organizations list-roots` then check policy types
- Check cross-account backup settings: `aws backup describe-region-settings`
- List backup policies: `aws organizations list-policies --filter BACKUP_POLICY`
- Check if a delegated administrator is configured: `aws organizations list-delegated-administrators --service-principal backup.amazonaws.com`

SHOULD:
- Describe the specific backup policy: `aws organizations describe-policy --policy-id <policy-id>`
- Check policy targets (OUs, accounts): `aws organizations list-targets-for-policy --policy-id <policy-id>`
- Verify the effective policy for a specific account: `aws organizations describe-effective-policy --policy-type BACKUP_POLICY --target-id <account-id>`
- Check if cross-account monitoring is enabled

MAY:
- Check CloudTrail in the management account for policy-related events
- Verify SCP policies are not blocking backup operations in member accounts
- Check if the member account has local backup plans that conflict with org policies

## Phase 2 — Remediate

MUST:
- Enable backup policy type in the organization if not enabled
- Enable cross-account backup and monitoring in AWS Backup settings (management account)
- Ensure backup policies are correctly attached to target OUs/accounts
- Verify the effective policy resolves correctly for member accounts

SHOULD:
- Use delegated administrator to avoid using the management account for daily operations
- Test backup policies on a single account before applying to the entire organization
- Ensure member accounts have the required IAM roles for backup operations

MAY:
- Implement backup policy inheritance with OU-level overrides
- Set up centralized backup monitoring in the management or delegated admin account
- Create compliance frameworks using Backup Audit Manager

## Common Issues

- symptoms: "Backup policies not taking effect in member accounts"
  diagnosis: "Backup policy type is not enabled in the organization, or policies are not attached to the correct OUs."
  resolution: "Enable BACKUP_POLICY type and attach policies to the correct OUs or accounts."

- symptoms: "Cross-account backup features unavailable"
  diagnosis: "Cross-account backup is not enabled in the management account's AWS Backup settings."
  resolution: "Enable cross-account backup in the management account: AWS Backup > Settings > Cross-account management."

- symptoms: "Member account cannot see organization backup jobs"
  diagnosis: "Cross-account monitoring is not enabled."
  resolution: "Enable cross-account monitoring in the management account's AWS Backup settings."

- symptoms: "Effective policy is empty or incorrect"
  diagnosis: "Policy inheritance is not resolving correctly due to conflicting policies at different OU levels."
  resolution: "Check policies at each OU level. Use @@assign, @@append, and @@remove operators correctly."

## Output Format

```yaml
root_cause: "organization_backup — <specific_cause>"
evidence:
  - type: organization
    content: "<organization and policy type status>"
  - type: backup_policy
    content: "<backup policy content and targets>"
  - type: effective_policy
    content: "<effective policy for the affected account>"
severity: HIGH
mitigation:
  immediate: "Fix organization backup policy configuration"
  long_term: "Implement centralized backup governance with delegated administrator"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable backup policy type in organization | RED | Organization-wide change — affects all member accounts |
| Enable cross-account backup and monitoring | RED | Organization-wide setting |
| Attach policies to target OUs/accounts | YELLOW | Changes backup behavior for targeted accounts |
| Verify effective policy | GREEN | Verification only |
| Use delegated administrator | YELLOW | Grants admin capabilities to member account |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Organization-level backup policy changes affect all member accounts
- Delegated administrator changes affect backup governance structure

## Data Sensitivity

- Organization backup policies reveal enterprise-wide data protection strategy
- Cross-account settings reveal organizational structure and trust relationships
- Effective policies reveal per-account backup requirements

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER disable organization backup policies without ensuring alternative coverage
- NEVER modify management account settings without change management approval

## Phase 3 — Rollback

- If backup policy type was enabled incorrectly: disable in organization (affects all accounts)
- If cross-account settings were changed: restore original settings in management account
- If policies were attached incorrectly: detach from wrong targets
- If delegated administrator was set incorrectly: remove delegation
- Document all organization-level changes for compliance audit trail

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
