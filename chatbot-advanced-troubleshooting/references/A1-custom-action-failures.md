---
title: "A1 — Custom Action Failures"
description: "Diagnose AWS Chatbot custom action execution failures including configuration, permissions, and runtime errors"
status: active
severity: HIGH
triggers: ["custom action failed", "action not working", "command not recognized", "action error in chat"]
owner: devops-agent
objective: "Resolve custom action execution failures in AWS Chatbot"
context: "Custom actions allow predefined commands to be run from chat. They require both channel configuration (action definition) and IAM role permissions. Missing either causes failures."
---

## Phase 1 — Triage

MUST:
- Get channel configuration: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>` or `aws chatbot get-microsoft-teams-channel-configuration --chat-configuration-arn <arn>`
- Check the channel IAM role permissions: `aws iam get-role --role-name <chatbot-role>`
- Verify the custom action is defined in the channel configuration

SHOULD:
- Check guardrail policy for action restrictions
- Review CloudTrail for Chatbot action events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 10`
- Verify the underlying AWS API call the action makes

MAY:
- Test the underlying AWS API call directly with the channel role
- Check Chatbot logs if logging is enabled

## Phase 2 — Remediate

MUST:
- Define the custom action in the channel configuration if missing
- Grant the channel IAM role permissions for the action's AWS API calls
- Ensure guardrails allow the custom action

SHOULD:
- Test the action with a non-production channel first
- Document custom actions and their required permissions
- Configure action response formatting for the target platform

MAY:
- Implement action approval workflows for sensitive operations
- Set up monitoring for custom action failures

## Safety Ratings

```
safety_ratings:
  - "describe-slack-channel-configurations: GREEN — read-only configuration listing"
  - "iam get-role: GREEN — read-only role metadata"
  - "update-slack-channel-configuration: YELLOW — modifies channel configuration"
  - "Custom action execution: YELLOW to RED — depends on the underlying AWS action"
```

## Escalation Conditions

- Custom action executes destructive AWS operations and fails mid-execution
- Action permissions require security team review
- Guardrail changes affect multiple channels and teams
- Action failure exposes sensitive information in chat

## Data Sensitivity

- `aws chatbot describe-slack-channel-configurations`: LOW — returns configuration metadata
- `aws iam get-role`: LOW — returns role metadata
- `aws iam list-attached-role-policies`: LOW — returns policy names
- Custom action outputs: MEDIUM to HIGH — may contain AWS resource details

## Prohibited Actions

- NEVER define custom actions that perform destructive operations without approval workflows
- NEVER grant the channel role AdministratorAccess to fix action permissions
- NEVER remove guardrails to make actions work without understanding security implications
- NEVER expose sensitive data (credentials, keys) through custom action responses

## Phase 3 — Rollback

1. If a custom action was misconfigured, update the channel configuration to remove or fix it
2. If IAM permissions were over-granted, revert to previous policy
3. If guardrails were modified, restore previous guardrail configuration
4. Verify other custom actions still work after changes
5. Document the change and notify channel users

## Common Issues

- symptoms: "Custom action returns 'command not recognized' in chat"
  diagnosis: "The action is not defined in the channel configuration."
  resolution: "Add the custom action definition to the channel configuration with the correct command template."

- symptoms: "Custom action returns 'access denied' error"
  diagnosis: "The channel IAM role lacks permissions for the action's underlying AWS API call."
  resolution: "Add the required IAM permissions to the channel role. Check both the role policy and guardrails."

- symptoms: "Custom action runs but returns empty or garbled response"
  diagnosis: "The action response is not formatted for the target platform or exceeds size limits."
  resolution: "Format responses for Slack (Block Kit) or Teams (Adaptive Cards). Keep responses concise."

## Output Format

```yaml
root_cause: "custom_action — <specific_cause>"
evidence:
  - type: action_error
    content: "<action failure details from chat or CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix action configuration or permissions"
  long_term: "Document and test custom actions before deployment"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
