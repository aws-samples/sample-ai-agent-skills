---
title: "B3 — Action Failures"
description: "Diagnose budget action execution failures"
status: active
severity: HIGH
triggers:
  - "budget action failed"
  - "budget action not executing"
  - "SCP not applied"
  - "budget automation failed"
owner: devops-agent
objective: "Resolve budget action configuration and execution issues"
context: "Budget actions can apply SCPs, IAM policies, or target EC2/RDS instances when thresholds are breached. Actions require an IAM role with appropriate permissions. Actions execute after budget evaluation, which has a delay."
---

## Phase 1 — Triage

MUST:
- Check budget action configuration: `aws budgets describe-budget-actions-for-budget --account-id <account-id> --budget-name <name>`
- Verify action execution status: `aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Check IAM role for budget actions: `aws iam get-role --role-name <role-name>`
- Verify the action type and target are correct

SHOULD:
- Check IAM role trust policy allows budgets.amazonaws.com to assume it
- Verify IAM role has permissions for the action type (organizations:AttachPolicy, iam:AttachUserPolicy, ec2:StopInstances)
- Review action execution history

MAY:
- Check CloudTrail for budget action API calls
- Verify SCP or IAM policy exists if action type is policy attachment

## Phase 2 — Remediate

MUST:
- Create/fix IAM role with correct trust policy for budgets.amazonaws.com
- Grant IAM role permissions for the specific action type
- Verify action target (SCP ID, IAM policy ARN, instance IDs) is correct

SHOULD:
- Test action with a low threshold to verify execution
- Set up approval workflow for destructive actions (instance stop)
- Monitor action execution with CloudTrail

MAY:
- Implement custom budget actions via SNS + Lambda for complex workflows
- Set up rollback procedures for budget actions

## Common Issues

- symptoms: "Budget action shows EXECUTION_FAILURE"
  diagnosis: "IAM role missing permissions or trust policy incorrect."
  resolution: "Fix IAM role trust policy and permissions for the action type."

- symptoms: "Action configured but never executes"
  diagnosis: "Budget threshold not exceeded or action approval pending."
  resolution: "Check budget actual vs threshold. Verify action approval status."

- symptoms: "SCP action fails in Organizations"
  diagnosis: "IAM role missing organizations:AttachPolicy permission."
  resolution: "Add organizations:AttachPolicy to the budget action IAM role."

## Output Format

```yaml
root_cause: "action_failure — <specific_cause>"
evidence:
  - type: action_config
    content: "<budget action configuration and status>"
  - type: iam_role
    content: "<IAM role trust policy and permissions>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role permissions and action configuration"
  long_term: "Implement action monitoring and approval workflows"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Budget actions may affect production resources |
| Blast Radius | HIGH — Budget actions can stop instances, apply SCPs, or restrict IAM |
| Rollback Complexity | MEDIUM — Reversing SCPs or restarting instances requires manual intervention |
| Compliance Scope | HIGH — Budget actions are financial governance controls |

## Escalation Conditions

- Escalate IMMEDIATELY if budget actions stop production instances unintentionally
- Escalate to security if SCP-based budget actions block critical operations
- Escalate to management if budget action failures leave spending uncontrolled

## Data Sensitivity

- **Budget action configurations** reveal financial control mechanisms
- **IAM role permissions** for budget actions should be tightly scoped
- **SCP and IAM policy details** are security-sensitive

## PROHIBITED Actions

- Do NOT configure budget actions to stop production instances without approval workflows
- Do NOT grant budget action IAM roles broader permissions than required
- Do NOT test budget actions in production without a rollback plan
- Do NOT disable budget actions without finance team approval

## Phase 3 — Rollback

- If SCP was incorrectly applied, detach it: `aws organizations detach-policy --policy-id <id> --target-id <target>`
- If instances were stopped, restart them: `aws ec2 start-instances --instance-ids <ids>`
- If IAM policy was attached, detach it from affected users/roles
- Restore budget action configuration to previous settings
- Document the incident for financial governance records

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
