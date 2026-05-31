---
title: "Z1 — General AWS Chatbot Troubleshooting (Catch-All)"
description: "Fallback SOP for Chatbot issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Chatbot issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks."
---

## Phase 1 — Triage

MUST:
- List Slack workspaces: `aws chatbot describe-slack-workspaces`
- List Slack channel configs: `aws chatbot describe-slack-channel-configurations`
- List Teams configs: `aws chatbot list-microsoft-teams-channel-configurations`
- Check channel IAM roles and SNS topics in configurations
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 20`

SHOULD:
- Verify SNS topic subscriptions for configured topics
- Check IAM role trust policies and permissions
- Test basic notification delivery

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Workspace/team authorization → Configuration (A1-A2)
- Notification delivery → Notifications (B1-B2)
- CLI command execution → Commands (C1-C2)
- IAM role/guardrail → IAM (D1-D2)
- CloudWatch/Security Hub → Integration (E1-E2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all channel configurations systematically
- Review SNS topic subscriptions and delivery status
- Verify IAM roles and guardrail policies
- Test notification and command functionality

SHOULD:
- Check AWS Health Dashboard for Chatbot service events
- Compare with known-good channel configurations
- Review recent CloudTrail events for configuration changes

## Phase 4 — Report

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<configuration|notifications|commands|iam|integration|unknown>"
investigation_path: "channels → SNS → IAM → <domain_classification>"
evidence:
  - type: channel_config
    content: "<channel configuration summary>"
  - type: sns_status
    content: "<SNS subscription and delivery status>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Classify failure domain and switch to specific SOP | GREEN |
| Check all channel configurations systematically | GREEN |
| Review SNS topic subscriptions and delivery status | GREEN |
| Verify IAM roles and guardrail policies | GREEN |
| Test notification and command functionality | GREEN |
| Check AWS Health Dashboard for Chatbot service events | GREEN |
| Compare with known-good channel configurations | GREEN |
| Review recent CloudTrail events for configuration changes | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- General troubleshooting may expose channel configurations, IAM roles, and notification content across all integrations

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. Rollback steps depend on the specific SOP identified during classification — follow the Phase 3 rollback of the matched SOP
2. If multiple changes were made during investigation, revert them in reverse order
3. Document all changes made during troubleshooting for audit purposes
4. Verify rollback by re-running the Phase 1 triage commands and confirming expected state

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
