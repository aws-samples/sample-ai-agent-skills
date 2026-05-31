---
title: "A1 — OpsItem Creation"
description: "Diagnose OpsItem creation failures"
status: active
severity: HIGH
triggers: ["cannot create OpsItem", "creation failed", "OpsItem error"]
owner: devops-agent
objective: "Resolve OpsItem creation issues"
context: "OpsItems can be created manually, via API, or through EventBridge rules. Creation failures are typically IAM or validation issues."
---

## Phase 1 — Triage

MUST:
- Verify OpsCenter is enabled: `aws ssm get-service-setting --setting-id /ssm/opsCenter`
- Check IAM permissions for ssm:CreateOpsItem
- Verify required fields (Title, Source, Description) are provided

SHOULD:
- Check service quotas for OpsItem limits
- Verify operational data format is valid JSON

MAY:
- Review CloudTrail for CreateOpsItem errors

## Phase 2 — Remediate

MUST:
- Enable OpsCenter if disabled
- Add ssm:CreateOpsItem permission to IAM policy
- Provide all required fields

SHOULD:
- Include dedup string for deduplication
- Add related resources for context

MAY:
- Use CloudFormation for OpsItem templates

## Common Issues

- symptoms: "CreateOpsItem returns OpsItemLimitExceededException"
  diagnosis: "Account has reached OpsItem limit."
  resolution: "Resolve or close existing OpsItems. Request quota increase if needed."

## Output Format

```yaml
root_cause: "opsitem_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Implement OpsItem lifecycle management"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — OpsItem creation is an operational tracking activity
- Service Disruption Risk: LOW — creation failures do not affect monitored resources
- Compliance Impact: LOW — OpsItem creation is an operational concern

## Escalation Conditions
- OpsItem creation fails with permission errors
- Automated OpsItem creation from EventBridge stops working
- OpsItem creation rate exceeds service quotas
- Critical operational events not generating OpsItems
- OpsItem data contains sensitive information that should be redacted

## Data Sensitivity
- OpsItem descriptions and operational data — INTERNAL
- Related resource ARNs — INTERNAL
- Notification configurations — INTERNAL
- Automation runbook references — INTERNAL
- Never include credentials or PII in OpsItem descriptions

## Prohibited Actions
- Do NOT include sensitive data (credentials, PII) in OpsItem descriptions
- Do NOT create OpsItems with overly broad related resource associations
- Do NOT bypass OpsItem severity classification standards
- Do NOT ignore OpsItem creation failures for critical events
- Do NOT grant OpsItem creation permissions to unauthorized users
- Do NOT use OpsItems as a substitute for proper incident management

## Phase 3 — Rollback
- Delete incorrectly created OpsItems
- Restore EventBridge rules for automated OpsItem creation
- Revert IAM permission changes that affected OpsItem creation
- Re-enable OpsItem notifications if disabled
- Verify OpsItem creation pipeline is functioning after changes
- Document OpsItem creation issues and resolutions

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
