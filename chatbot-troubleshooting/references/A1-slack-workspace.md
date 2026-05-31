---
title: "A1 — AWS Chatbot Slack Workspace Configuration Issues"
description: "Diagnose Slack workspace authorization and channel configuration problems"
status: active
severity: HIGH
triggers:
  - "Slack workspace"
  - "Slack configuration"
  - "Slack authorization"
  - "Slack channel not working"
  - "Slack app"
owner: devops-agent
objective: "Identify and resolve Slack workspace and channel configuration issues for AWS Chatbot"
context: "AWS Chatbot Slack integration requires workspace authorization (OAuth), channel configuration with IAM role and SNS topics, and the Chatbot Slack app installed. Issues stem from authorization failures, app removal, or misconfigured channels."
---

## Phase 1 — Triage

MUST:
- List configured Slack workspaces: `aws chatbot describe-slack-workspaces`
- List Slack channel configurations: `aws chatbot describe-slack-channel-configurations`
- Verify workspace is authorized (appears in workspace list)
- Check specific channel config: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>`
- Check CloudTrail for Chatbot events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 10`

SHOULD:
- Verify the Chatbot Slack app is installed in the workspace
- Check channel IAM role exists: `aws iam get-role --role-name <chatbot-role>`
- Verify SNS topics are configured in the channel configuration

MAY:
- Check Slack workspace admin settings for app restrictions
- Verify OAuth token hasn't been revoked

## Phase 2 — Remediate

MUST:
- Authorize workspace in AWS Console if not listed (cannot be done via CLI)
- Install the AWS Chatbot Slack app in the workspace
- Configure channel with valid IAM role and SNS topic ARNs

SHOULD:
- Use a dedicated IAM role per channel for least privilege
- Subscribe required SNS topics to the channel
- Test notification delivery after configuration

MAY:
- Set up multiple channels for different notification types
- Configure guardrail policies for CLI command access

## Common Issues

- symptoms: "Workspace not appearing in Chatbot console"
  diagnosis: "Workspace authorization not completed or OAuth token expired."
  resolution: "Re-authorize the workspace in the AWS Console."

- symptoms: "Channel configured but no notifications received"
  diagnosis: "SNS topics not subscribed or Slack app removed from workspace."
  resolution: "Verify SNS subscriptions and reinstall the Chatbot Slack app."

- symptoms: "Chatbot app removed from Slack workspace"
  diagnosis: "Workspace admin removed or restricted the AWS Chatbot app."
  resolution: "Reinstall the AWS Chatbot app and re-authorize the workspace."

## Output Format

```yaml
root_cause: "slack_workspace_issue — <specific_cause>"
evidence:
  - type: workspace_config
    content: "<workspace authorization status>"
  - type: channel_config
    content: "<channel configuration details>"
  - type: slack_app
    content: "<Slack app installation status>"
severity: HIGH
mitigation:
  immediate: "Re-authorize workspace or fix channel configuration"
  long_term: "Document workspace authorization procedures and monitor app status"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Authorize workspace in AWS Console | GREEN |
| Install the AWS Chatbot Slack app in the workspace | GREEN |
| Configure channel with valid IAM role and SNS topic ARNs | YELLOW |
| Use a dedicated IAM role per channel for least privilege | GREEN |
| Subscribe required SNS topics to the channel | GREEN |
| Test notification delivery after configuration | GREEN |
| Set up multiple channels for different notification types | GREEN |
| Configure guardrail policies for CLI command access | YELLOW |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- Slack workspace authorization tokens and channel configurations may expose organizational communication structure

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If workspace authorization was changed, re-authorize or remove the workspace in the AWS Console
2. If channel configuration was modified, delete and recreate with previous settings: `aws chatbot delete-slack-channel-configuration --chat-configuration-arn <arn>`
3. If IAM roles were modified, revert to the previous policy version
4. If SNS topic subscriptions were changed, restore previous subscription configuration
5. Verify rollback by confirming `aws chatbot describe-slack-channel-configurations` shows expected state

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
