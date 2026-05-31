---
title: "C2 — AWS Chatbot Command Permission Issues"
description: "Diagnose permission errors when running CLI commands in chat"
status: active
severity: MEDIUM
triggers:
  - "permission denied in chat"
  - "command not allowed"
  - "guardrail blocked"
  - "access denied command"
owner: devops-agent
objective: "Identify and resolve permission issues for CLI commands in AWS Chatbot"
context: "CLI command permissions are controlled by two layers: the channel IAM role (maximum permissions) and guardrail policies (additional restrictions). Both must allow an action for it to succeed."
---

## Phase 1 — Triage

MUST:
- Check channel IAM role: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>` (look at iamRoleArn)
- Check guardrail policies: look at guardrailPolicies in channel configuration
- Verify IAM role permissions: `aws iam list-attached-role-policies --role-name <role-name>`
- Check guardrail policy content: `aws iam get-policy-version --policy-arn <guardrail-policy-arn> --version-id <version>`
- Determine if the command is read-only or mutating

SHOULD:
- Simulate the specific action: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names <service>:<action>`
- Check if the guardrail policy explicitly denies the action
- Verify the effective permission (intersection of role and guardrail)

MAY:
- Check CloudTrail for denied command attempts
- Compare with a working channel configuration

## Phase 2 — Remediate

MUST:
- Add required permissions to the channel IAM role
- Update guardrail policy to allow the specific action (for mutating commands)
- Ensure both role AND guardrail allow the action

SHOULD:
- Use separate channels with different permission levels
- Create custom guardrail policies for specific use cases
- Document which commands are allowed in which channels

MAY:
- Implement approval workflows for sensitive commands
- Set up audit logging for command execution

## Common Issues

- symptoms: "Read-only command works but mutating command fails"
  diagnosis: "Default ReadOnlyAccess guardrail policy blocks mutating commands."
  resolution: "Create a custom guardrail policy that allows the specific mutating actions."

- symptoms: "All commands fail with permission error"
  diagnosis: "Channel IAM role has no permissions or trust policy is wrong."
  resolution: "Verify IAM role trust policy includes chatbot.amazonaws.com and add required permissions."

- symptoms: "Command works in one channel but not another"
  diagnosis: "Different channels have different IAM roles or guardrail policies."
  resolution: "Check the specific channel's IAM role and guardrail policy configuration."

## Output Format

```yaml
root_cause: "command_permission_issue — <specific_cause>"
evidence:
  - type: iam_role
    content: "<role permissions>"
  - type: guardrail_policy
    content: "<guardrail policy content>"
  - type: effective_permission
    content: "<intersection of role and guardrail>"
severity: MEDIUM
mitigation:
  immediate: "Update IAM role or guardrail policy"
  long_term: "Implement channel-level permission documentation and auditing"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Add required permissions to the channel IAM role | YELLOW |
| Update guardrail policy to allow the specific action | RED |
| Ensure both role AND guardrail allow the action | YELLOW |
| Use separate channels with different permission levels | GREEN |
| Create custom guardrail policies for specific use cases | YELLOW |
| Document which commands are allowed in which channels | GREEN |
| Implement approval workflows for sensitive commands | GREEN |
| Set up audit logging for command execution | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- Permission configurations reveal the scope of actions available through chat interfaces

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If IAM role permissions were broadened, revert to the previous policy version
2. If guardrail policy was modified, restore the previous guardrail policy
3. If new channels were created with elevated permissions, delete them
4. Verify rollback by running `aws iam simulate-principal-policy` against the channel role

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
