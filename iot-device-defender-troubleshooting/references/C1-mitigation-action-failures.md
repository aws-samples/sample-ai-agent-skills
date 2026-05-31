---
title: "C1 — Mitigation Action Failures"
description: "Diagnose mitigation action configuration and execution failures"
status: active
severity: HIGH
triggers: ["mitigation action failed", "cannot apply mitigation", "action execution error", "mitigation task error"]
owner: devops-agent
objective: "Resolve mitigation action issues including IAM permissions, action configuration, and execution failures"
context: "Mitigation actions are pre-defined action types applied to audit findings. They require IAM roles with specific permissions. Actions include adding things to groups, enabling logging, publishing to SNS, and updating certificates."
---

## Phase 1 — Triage

MUST:
- List mitigation actions: `aws iot list-mitigation-actions`
- Check action details: `aws iot describe-mitigation-action --action-name <name>`
- Check mitigation task status: `aws iot describe-audit-mitigation-actions-task --task-id <task-id>`

SHOULD:
- Verify IAM role permissions for the mitigation action
- Check if the target resource exists and is accessible
- Review task execution results for specific finding failures

MAY:
- Check CloudTrail for mitigation action API errors
- Verify SNS topic permissions if using PUBLISH_FINDING_TO_SNS action

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for the mitigation action type
- Ensure target resources exist (thing groups, SNS topics, etc.)
- Retry failed mitigation tasks after fixing the root cause

SHOULD:
- Test mitigation actions on non-production resources first
- Verify the action type matches the finding type

MAY:
- Create multiple mitigation actions for different finding types
- Set up automated mitigation for specific audit checks

## Safety Ratings

```
safety_ratings:
  - "iot list-mitigation-actions: GREEN — read-only listing"
  - "iot describe-mitigation-action: GREEN — read-only action details"
  - "iot start-audit-mitigation-actions-task: YELLOW — applies actions to resources"
  - "ADD_THINGS_TO_THING_GROUP: YELLOW — moves devices to quarantine group"
```

## Escalation Conditions

- Mitigation actions fail on critical findings requiring immediate remediation
- IAM role for mitigation is compromised or deleted
- Mitigation action causes unintended side effects on production devices
- Large number of findings require mitigation but actions fail consistently

## Data Sensitivity

- `aws iot list-mitigation-actions`: LOW — returns action names and types
- `aws iot describe-mitigation-action`: MEDIUM — contains action configuration and IAM role
- `mitigation task results`: MEDIUM — reveals which resources were modified
- `IAM role for mitigation`: MEDIUM — role ARN and permissions

## Prohibited Actions

- NEVER apply mitigation actions to production devices without testing
- NEVER use REPLACE_DEFAULT_POLICY_VERSION without reviewing the replacement policy
- NEVER delete mitigation actions that are referenced by automated workflows
- NEVER apply UPDATE_DEVICE_CERTIFICATE (revoke) without confirming device can be re-provisioned

## Phase 3 — Rollback

1. If ADD_THINGS_TO_THING_GROUP moved devices incorrectly, remove them from the group
2. If REPLACE_DEFAULT_POLICY_VERSION changed the wrong policy, restore previous version
3. If UPDATE_DEVICE_CERTIFICATE revoked certificates, re-activate: `aws iot update-certificate --certificate-id <id> --new-status ACTIVE`
4. Verify affected devices can still connect and operate
5. Document the mitigation action failure and corrective steps

## Common Issues

- symptoms: "Mitigation task fails — AccessDeniedException on iot:AddThingToThingGroup"
  diagnosis: "IAM role for mitigation action lacks required permissions."
  resolution: "Add iot:AddThingToThingGroup permission to the mitigation action IAM role."

## Output Format

```yaml
root_cause: "mitigation_action — <specific_cause>"
evidence:
  - type: mitigation_error
    content: "<error details from task execution>"
severity: HIGH
mitigation:
  immediate: "Fix mitigation action configuration"
  long_term: "Test mitigation actions and automate for recurring findings"
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
