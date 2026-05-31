---
title: "A2 — AWS Chatbot Microsoft Teams Channel Issues"
description: "Diagnose Microsoft Teams channel configuration and tenant consent problems"
status: active
severity: HIGH
triggers:
  - "Teams channel"
  - "Teams configuration"
  - "Teams not working"
  - "tenant consent"
  - "Azure AD"
owner: devops-agent
objective: "Identify and resolve Microsoft Teams channel configuration issues for AWS Chatbot"
context: "AWS Chatbot Teams integration requires Azure AD tenant admin consent, team configuration, and channel setup with IAM role and SNS topics. Issues stem from missing tenant consent, team authorization, or channel misconfiguration."
---

## Phase 1 — Triage

MUST:
- List configured Teams: `aws chatbot list-microsoft-teams-configured-teams`
- List Teams channel configurations: `aws chatbot list-microsoft-teams-channel-configurations`
- Verify tenant/team is configured
- Check specific channel config details
- Check CloudTrail for Chatbot events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 10`

SHOULD:
- Verify Azure AD tenant admin consent was granted
- Check channel IAM role: `aws iam get-role --role-name <chatbot-role>`
- Verify SNS topics are configured

MAY:
- Check Azure AD app registrations for AWS Chatbot
- Verify Teams channel permissions

## Phase 2 — Remediate

MUST:
- Obtain Azure AD tenant admin consent for the AWS Chatbot app
- Configure the team in AWS Chatbot console
- Set up channel configuration with IAM role and SNS topics

SHOULD:
- Use a dedicated IAM role per channel
- Test notification delivery after configuration
- Document the tenant consent process for future reference

MAY:
- Configure multiple channels for different notification types
- Set up guardrail policies for CLI commands

## Common Issues

- symptoms: "Cannot configure Teams channel — tenant not authorized"
  diagnosis: "Azure AD tenant admin consent not granted for AWS Chatbot app."
  resolution: "Request Azure AD tenant admin to grant consent for the AWS Chatbot app."

- symptoms: "Teams configured but notifications not appearing"
  diagnosis: "SNS topics not subscribed or channel configuration incorrect."
  resolution: "Verify SNS subscriptions and channel configuration."

- symptoms: "Teams integration stopped working"
  diagnosis: "Azure AD consent revoked or Teams app permissions changed."
  resolution: "Re-grant Azure AD tenant consent and verify app permissions."

## Output Format

```yaml
root_cause: "teams_channel_issue — <specific_cause>"
evidence:
  - type: teams_config
    content: "<team and channel configuration>"
  - type: tenant_consent
    content: "<Azure AD consent status>"
  - type: sns_subscriptions
    content: "<SNS topic subscriptions>"
severity: HIGH
mitigation:
  immediate: "Fix tenant consent or channel configuration"
  long_term: "Document Teams setup procedures and monitor consent status"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Obtain Azure AD tenant admin consent for the AWS Chatbot app | YELLOW |
| Configure the team in AWS Chatbot console | GREEN |
| Set up channel configuration with IAM role and SNS topics | YELLOW |
| Use a dedicated IAM role per channel | GREEN |
| Test notification delivery after configuration | GREEN |
| Document the tenant consent process for future reference | GREEN |
| Configure multiple channels for different notification types | GREEN |
| Set up guardrail policies for CLI commands | YELLOW |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- Azure AD tenant consent and Teams channel configurations may expose organizational identity infrastructure

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If Azure AD consent was granted in error, revoke it through Azure AD admin portal
2. If Teams channel configuration was modified, delete and recreate with previous settings
3. If IAM roles were modified, revert to the previous policy version
4. If SNS topic subscriptions were changed, restore previous configuration
5. Verify rollback by confirming `aws chatbot list-microsoft-teams-channel-configurations` shows expected state

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
