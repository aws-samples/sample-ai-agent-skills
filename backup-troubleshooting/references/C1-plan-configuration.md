---
title: "C1 — Backup Plan Configuration Issues"
description: "Diagnose misconfigured backup plans that cause unexpected behavior"
status: active
severity: MEDIUM
triggers:
  - "backup plan wrong"
  - "plan misconfigured"
  - "backup plan not working"
  - "plan configuration"
  - "backup rule"
owner: devops-agent
objective: "Identify and fix backup plan configuration issues causing unexpected backup behavior"
context: "Backup plans define rules with schedules, lifecycle policies, target vaults, and copy actions. Misconfiguration can lead to missed backups, incorrect retention, wrong vault targets, or excessive costs. Plans can have up to 10 rules, each with independent settings."
---

## Phase 1 — Triage

MUST:
- Get the backup plan details: `aws backup get-backup-plan --backup-plan-id <plan-id>`
- Review each rule's schedule expression, lifecycle, and target vault
- List backup selections: `aws backup list-backup-selections --backup-plan-id <plan-id>`
- Get selection details: `aws backup get-backup-selection --backup-plan-id <plan-id> --selection-id <selection-id>`
- Check plan version history: `aws backup list-backup-plan-versions --backup-plan-id <plan-id>`

SHOULD:
- Verify lifecycle rules are valid (cold storage minimums, delete-after constraints)
- Check if multiple rules have overlapping schedules causing duplicate backups
- Verify the target vault exists and is accessible for each rule
- Check if copy actions are configured correctly (destination vault, KMS key)

MAY:
- Compare with previous plan versions to identify recent changes
- Check if the plan is associated with the correct AWS account and region
- Review backup plan templates for best practices

## Phase 2 — Remediate

MUST:
- Fix identified configuration issues (lifecycle, schedule, vault, selections)
- Update the backup plan: `aws backup update-backup-plan --backup-plan-id <plan-id> --backup-plan <plan-json>`
- Verify the updated plan by describing it again

SHOULD:
- Test the updated plan with an on-demand backup
- Ensure lifecycle rules comply with vault lock requirements
- Review and update backup selections if resource targeting is incorrect

MAY:
- Use AWS Backup Audit Manager to validate plan compliance
- Create plan templates for standardized backup configurations

## Common Issues

- symptoms: "Backups going to the wrong vault"
  diagnosis: "The backup rule's TargetBackupVaultName points to an unintended vault."
  resolution: "Update the rule's TargetBackupVaultName to the correct vault."

- symptoms: "Recovery points not transitioning to cold storage"
  diagnosis: "The resource type does not support cold storage, or MoveToColdStorageAfterDays is not set."
  resolution: "Verify the resource type supports cold storage (EBS, EFS, DynamoDB only). Set the lifecycle."

- symptoms: "Backup plan creates duplicate backups"
  diagnosis: "Multiple rules with overlapping schedules target the same resources."
  resolution: "Consolidate rules or adjust schedules to avoid overlap."

- symptoms: "Lifecycle validation error when updating plan"
  diagnosis: "DeleteAfterDays is less than 90 days greater than MoveToColdStorageAfterDays."
  resolution: "Ensure DeleteAfterDays >= MoveToColdStorageAfterDays + 90."

## Output Format

```yaml
root_cause: "plan_configuration — <specific_cause>"
evidence:
  - type: backup_plan
    content: "<plan rules and configuration>"
  - type: selections
    content: "<resource selections>"
  - type: lifecycle
    content: "<lifecycle rule details>"
severity: MEDIUM
mitigation:
  immediate: "Fix the plan configuration and verify"
  long_term: "Implement plan templates and compliance monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix lifecycle, schedule, vault, selections | YELLOW | Changes backup behavior — review each change |
| Update backup plan | YELLOW | Modifies backup plan — affects all targeted resources |
| Verify updated plan | GREEN | Verification only |
| Test with on-demand backup | GREEN | Testing — non-destructive |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Backup plan changes affect all resources in the plan's selections
- Lifecycle changes must comply with vault lock retention requirements

## Data Sensitivity

- Backup plan configuration reveals data protection strategy and scope
- Lifecycle rules reveal retention policies and compliance requirements
- Selection criteria reveal which resources are protected

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER set DeleteAfterDays below vault lock MinRetentionDays
- NEVER remove backup rules without ensuring alternative coverage exists

## Phase 3 — Rollback

- If backup plan was updated incorrectly: restore previous plan version using `aws backup update-backup-plan`
- If lifecycle was changed: restore original lifecycle settings
- If selections were modified: restore original selection criteria
- Document all backup plan changes for compliance audit trail

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
