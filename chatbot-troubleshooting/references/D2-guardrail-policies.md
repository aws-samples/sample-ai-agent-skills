---
title: "D2 — AWS Chatbot Guardrail Policy Issues"
description: "Diagnose guardrail policy configuration and restriction problems"
status: active
severity: MEDIUM
triggers:
  - "guardrail policy"
  - "guardrail blocking"
  - "policy restriction"
  - "command blocked by guardrail"
owner: devops-agent
objective: "Identify and resolve guardrail policy issues affecting CLI command execution"
context: "Guardrail policies are IAM policies that act as an additional permission boundary on top of the channel IAM role. They restrict what actions can be performed via CLI commands in chat. The effective permission is the intersection of the role and guardrail."
---

## Phase 1 — Triage

MUST:
- Check guardrail policies on channel: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>` (look at guardrailPolicies)
- Get guardrail policy content: `aws iam get-policy --policy-arn <guardrail-arn>`
- Get policy version: `aws iam get-policy-version --policy-arn <guardrail-arn> --version-id <version>`
- Determine if the blocked action is read-only or mutating
- Check if default ReadOnlyAccess guardrail is in use

SHOULD:
- Compare guardrail policy with channel IAM role permissions
- Identify the specific action being blocked
- Check if guardrail has explicit deny statements

MAY:
- Review all guardrail policies if multiple are attached
- Compare with other channel configurations that work

## Phase 2 — Remediate

MUST:
- For mutating commands, create a custom guardrail policy that allows the specific actions
- Ensure guardrail policy does not explicitly deny required actions
- Update channel configuration with the new guardrail policy

SHOULD:
- Keep guardrail policies as restrictive as possible
- Use separate guardrail policies for different channel types
- Document which actions each guardrail allows

MAY:
- Create guardrail policy templates for common use cases
- Implement guardrail policy review process

## Common Issues

- symptoms: "Mutating command blocked despite IAM role having permissions"
  diagnosis: "Default ReadOnlyAccess guardrail blocks all mutating commands."
  resolution: "Create a custom guardrail policy that allows the specific mutating actions."

- symptoms: "Guardrail policy allows action but command still fails"
  diagnosis: "Channel IAM role doesn't have the permission (guardrail only allows, doesn't grant)."
  resolution: "Add the permission to both the IAM role AND the guardrail policy."

- symptoms: "All commands blocked by guardrail"
  diagnosis: "Guardrail policy has an explicit deny on all actions or is empty."
  resolution: "Review and fix the guardrail policy to allow intended actions."

## Output Format

```yaml
root_cause: "guardrail_policy_issue — <specific_cause>"
evidence:
  - type: guardrail_policy
    content: "<policy content and allowed/denied actions>"
  - type: iam_role
    content: "<channel role permissions>"
  - type: effective_permission
    content: "<intersection analysis>"
severity: MEDIUM
mitigation:
  immediate: "Update guardrail policy to allow required actions"
  long_term: "Create standardized guardrail policy templates"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| For mutating commands, create a custom guardrail policy | YELLOW |
| Ensure guardrail policy does not explicitly deny required actions | YELLOW |
| Update channel configuration with the new guardrail policy | YELLOW |
| Keep guardrail policies as restrictive as possible | GREEN |
| Use separate guardrail policies for different channel types | GREEN |
| Document which actions each guardrail allows | GREEN |
| Create guardrail policy templates for common use cases | GREEN |
| Implement guardrail policy review process | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- Guardrail policies define the security boundary for chat-based command execution

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If guardrail policy was modified, revert to the previous policy version: `aws iam create-policy-version --policy-arn <arn> --policy-document <previous-doc> --set-as-default`
2. If channel configuration was updated with a new guardrail, restore the previous guardrail ARN
3. If a new guardrail policy was created, delete it after reverting channel configurations
4. Verify rollback by confirming channel guardrail policy matches expected configuration

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
