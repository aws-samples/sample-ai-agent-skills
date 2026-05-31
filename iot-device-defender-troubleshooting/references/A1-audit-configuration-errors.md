---
title: "A1 — Audit Configuration Errors"
description: "Diagnose IoT Device Defender audit configuration issues"
status: active
severity: HIGH
triggers: ["audit not running", "audit configuration error", "cannot enable audit", "audit check disabled"]
owner: devops-agent
objective: "Resolve audit configuration issues including check enablement, scheduling, and IAM permissions"
context: "Device Defender audit checks IoT account configuration against security best practices. Checks must be enabled individually and can run on a schedule or on-demand."
---

## Phase 1 — Triage

MUST:
- Check audit configuration: `aws iot describe-account-audit-configuration`
- List scheduled audits: `aws iot list-scheduled-audits`
- Verify audit IAM role: `aws iot describe-account-audit-configuration --query 'roleArn'`

SHOULD:
- Check which audit checks are enabled: `aws iot describe-account-audit-configuration --query 'auditCheckConfigurations'`
- Verify IAM role has required permissions for audit operations
- Review recent audit task results: `aws iot list-audit-tasks --task-status COMPLETED --start-time <7-days-ago> --end-time <now>`

MAY:
- Check CloudTrail for UpdateAccountAuditConfiguration errors
- Verify SNS topic for audit notifications

## Phase 2 — Remediate

MUST:
- Enable required audit checks: `aws iot update-account-audit-configuration --audit-check-configurations '{"<CHECK_NAME>":{"enabled":true}}'`
- Fix IAM role permissions for audit operations
- Configure audit notification target if notifications are needed

SHOULD:
- Schedule daily audits: `aws iot create-scheduled-audit --scheduled-audit-name daily-audit --frequency DAILY --target-check-names <checks>`
- Enable all recommended audit checks for comprehensive coverage

MAY:
- Configure audit suppression for accepted risks
- Set up CloudWatch alarms for audit failures

## Safety Ratings

```
safety_ratings:
  - "iot describe-account-audit-configuration: GREEN — read-only configuration check"
  - "iot list-scheduled-audits: GREEN — read-only listing"
  - "iot update-account-audit-configuration: YELLOW — modifies audit configuration"
  - "iot create-scheduled-audit: YELLOW — creates recurring audit schedule"
```

## Escalation Conditions

- Audit configuration cannot be updated due to IAM permission issues
- Audit tasks fail consistently with internal service errors
- Required audit checks are not available in the region
- Audit IAM role is compromised or deleted

## Data Sensitivity

- `aws iot describe-account-audit-configuration`: LOW — returns audit configuration
- `aws iot list-scheduled-audits`: LOW — returns schedule metadata
- `audit IAM role`: MEDIUM — role ARN reveals account structure
- `audit findings`: MEDIUM — reveal security configuration weaknesses

## Prohibited Actions

- NEVER disable all audit checks to suppress findings
- NEVER delete the audit IAM role while audits are scheduled
- NEVER configure audit notifications to public SNS topics
- NEVER ignore audit configuration errors — they indicate security monitoring gaps

## Phase 3 — Rollback

1. If audit configuration was changed incorrectly, restore previous check settings
2. If scheduled audit was created with wrong parameters, delete and recreate: `aws iot delete-scheduled-audit --scheduled-audit-name <name>`
3. If IAM role was modified, restore previous policy
4. Verify audit is running correctly: `aws iot start-on-demand-audit-task --target-check-names <checks>`
5. Document the configuration change and verify findings are generated

## Common Issues

- symptoms: "Audit not running — no recent audit tasks"
  diagnosis: "No scheduled audit configured and no on-demand audits triggered."
  resolution: "Create a scheduled audit or run on-demand: aws iot start-on-demand-audit-task."

## Output Format

```yaml
root_cause: "audit_configuration — <specific_cause>"
evidence:
  - type: audit_config_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix audit configuration"
  long_term: "Schedule daily audits with all recommended checks"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
