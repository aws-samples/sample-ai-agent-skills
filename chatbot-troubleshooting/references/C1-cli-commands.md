---
title: "C1 — AWS Chatbot CLI Commands in Chat Issues"
description: "Diagnose CLI command execution failures in Slack/Teams"
status: active
severity: MEDIUM
triggers:
  - "CLI command failed"
  - "@aws command"
  - "command not working"
  - "command error in chat"
  - "run command in Slack"
owner: devops-agent
objective: "Identify and resolve CLI command execution issues in AWS Chatbot"
context: "AWS Chatbot allows running AWS CLI commands in Slack/Teams using @aws prefix. Commands are subject to channel IAM role permissions and guardrail policies. Not all commands are supported."
---

## Phase 1 — Triage

MUST:
- Check channel configuration for CLI support: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>`
- Verify channel IAM role: `aws iam get-role --role-name <chatbot-role>`
- Check guardrail policies: look at guardrailPolicies in channel configuration
- Verify the command syntax is correct (@aws <service> <command>)
- Check if the command is a supported Chatbot command

SHOULD:
- Check IAM role permissions for the specific command: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names <service>:<action>`
- Verify guardrail policy allows the action
- Check if the command is read-only or mutating

MAY:
- Check CloudTrail for command execution events
- Test with a simple read-only command first (e.g., @aws sts get-caller-identity)

## Phase 2 — Remediate

MUST:
- Use correct syntax: `@aws <service> <command> [parameters]`
- Ensure channel IAM role has permissions for the command
- For mutating commands, update guardrail policy to allow the action

SHOULD:
- Start with read-only commands to verify basic functionality
- Use the default ReadOnlyAccess guardrail for read-only channels
- Create custom guardrail policies for channels that need mutating access

MAY:
- Document supported commands for team reference
- Set up separate channels for read-only vs mutating commands

## Common Issues

- symptoms: "Command returns 'not authorized' error"
  diagnosis: "Channel IAM role lacks permissions for the command."
  resolution: "Add required permissions to the channel IAM role."

- symptoms: "Mutating command blocked"
  diagnosis: "Guardrail policy only allows read-only commands (default)."
  resolution: "Update guardrail policy to allow the specific mutating action."

- symptoms: "Command syntax not recognized"
  diagnosis: "Incorrect command syntax or unsupported command."
  resolution: "Use @aws <service> <command> syntax. Check supported commands list."

## Output Format

```yaml
root_cause: "cli_command_issue — <specific_cause>"
evidence:
  - type: channel_config
    content: "<IAM role and guardrail policies>"
  - type: permission_test
    content: "<IAM role permission check>"
  - type: command_details
    content: "<command syntax and type>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM role permissions or guardrail policy"
  long_term: "Document supported commands and permission requirements"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use correct syntax: @aws service command parameters | GREEN |
| Ensure channel IAM role has permissions for the command | YELLOW |
| For mutating commands, update guardrail policy to allow the action | RED |
| Start with read-only commands to verify basic functionality | GREEN |
| Use the default ReadOnlyAccess guardrail for read-only channels | GREEN |
| Create custom guardrail policies for channels that need mutating access | YELLOW |
| Document supported commands for team reference | GREEN |
| Set up separate channels for read-only vs mutating commands | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- CLI command output may contain sensitive AWS resource information

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If guardrail policy was modified to allow mutating commands, revert to the previous policy version
2. If IAM role permissions were broadened, revert to the previous policy
3. If channel configuration was changed, restore previous settings
4. Verify rollback by testing that previously blocked commands remain blocked

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
