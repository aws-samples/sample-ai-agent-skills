---
title: "F1 — Account-Level Settings"
description: "Diagnose account-level SSM settings affecting Fleet Manager behavior"
status: active
severity: MEDIUM
triggers: ["SSM settings wrong", "managed instance tier", "default host management", "account settings"]
owner: devops-agent
objective: "Resolve account-level SSM settings issues affecting Fleet Manager"
context: "Account-level settings control SSM behavior including managed instance tier (standard/advanced), default host management configuration, and SSM Agent auto-update. These settings affect all instances in the account and region."
---

## Phase 1 — Triage

MUST:
- Check managed instance tier: `aws ssm get-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/activation-tier`
- Check default host management: `aws ssm get-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/default-ec2-instance-management-role`
- Check SSM Agent auto-update: `aws ssm get-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/auto-update-ssm-agent`

SHOULD:
- Verify the account ID and region are correct for the settings query
- Check if settings were recently changed via CloudTrail

MAY:
- Compare settings across regions for consistency
- Review organization-level SSM policies if applicable

## Phase 2 — Remediate

MUST:
- Update tier if on-premises limit reached: `aws ssm update-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/activation-tier --setting-value advanced`
- Configure default host management role if needed: `aws ssm update-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/default-ec2-instance-management-role --setting-value <role-name>`
- Enable auto-update if agents are outdated: `aws ssm update-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/auto-update-ssm-agent --setting-value true`

SHOULD:
- Document the reason for any setting changes
- Apply consistent settings across all active regions

MAY:
- Use AWS Organizations to enforce SSM settings via SCPs
- Automate settings configuration via CloudFormation or Terraform

## Common Issues

- symptoms: "Cannot register more on-premises instances — limit reached"
  diagnosis: "Standard tier has 1,000 on-premises instance limit per region."
  resolution: "Upgrade to advanced tier. Note: advanced tier incurs per-instance charges."

- symptoms: "EC2 instances not automatically managed without instance profiles"
  diagnosis: "Default Host Management Configuration not enabled."
  resolution: "Enable default host management with an appropriate IAM role."

## Safety Ratings

```
safety_ratings:
  - "get-service-setting: GREEN — read-only setting query"
  - "update-service-setting activation-tier: YELLOW — may incur additional costs"
  - "update-service-setting default-role: YELLOW — changes default IAM behavior for all EC2 instances"
  - "update-service-setting auto-update: YELLOW — enables automatic agent updates fleet-wide"
```

## Escalation Conditions

- Tier upgrade requires budget approval due to per-instance costs
- Default host management role needs security review before enabling
- Settings changes affect multiple teams or environments in the same account
- Organization-level policies conflict with account-level settings

## Data Sensitivity

- `aws ssm get-service-setting`: LOW — returns setting values only
- `aws ssm update-service-setting`: MEDIUM — modifies account-wide behavior
- `aws cloudtrail lookup-events`: MEDIUM — reveals who changed settings and when
- Account-level settings: MEDIUM — affect all instances in the account and region

## Prohibited Actions

- NEVER upgrade to advanced tier without cost approval — it charges per on-premises instance
- NEVER enable default host management with an overly permissive IAM role
- NEVER disable SSM Agent auto-update on accounts with security patching requirements
- NEVER change account-level settings without notifying all teams using SSM in the account

## Phase 3 — Rollback

1. Revert tier setting: `aws ssm reset-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/activation-tier`
2. Revert default host management: `aws ssm reset-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/default-ec2-instance-management-role`
3. Revert auto-update setting: `aws ssm reset-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/auto-update-ssm-agent`
4. Verify instances are not affected by the rollback
5. Document the rollback and communicate to affected teams

## Output Format

```yaml
root_cause: "account_level_settings — <specific_cause>"
evidence:
  - type: settings_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Correct account-level setting"
  long_term: "Enforce settings via IaC and organizational policies"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
