---
title: "E2 — AWS Chatbot Security Hub Integration Issues"
description: "Diagnose Security Hub finding notification delivery to Chatbot channels"
status: active
severity: MEDIUM
triggers:
  - "Security Hub notification"
  - "findings not in chat"
  - "Security Hub Chatbot"
  - "security alert missing"
owner: devops-agent
objective: "Identify and resolve Security Hub finding notification delivery issues to Chatbot channels"
context: "Security Hub findings can be sent to Chatbot via EventBridge rules targeting SNS topics. The EventBridge rule must match Security Hub finding events and target the SNS topic configured in the Chatbot channel."
---

## Phase 1 — Triage

MUST:
- Check EventBridge rules for Security Hub: `aws events list-rules --query 'Rules[?contains(Name, \`SecurityHub\`) || contains(Name, \`securityhub\`)]'`
- Check rule event pattern: `aws events describe-rule --name <rule-name> --query 'EventPattern'`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify target SNS topic is in Chatbot channel configuration
- Check Security Hub is enabled: `aws securityhub describe-hub`

SHOULD:
- Verify EventBridge rule is ENABLED
- Check if findings are being generated: `aws securityhub get-findings --max-results 5`
- Verify SNS topic subscription to Chatbot

MAY:
- Check EventBridge invocation metrics
- Test with a custom Security Hub finding import

## Phase 2 — Remediate

MUST:
- Create EventBridge rule with Security Hub finding event pattern:
  ```json
  {"source": ["aws.securityhub"], "detail-type": ["Security Hub Findings - Imported"]}
  ```
- Set rule target to the SNS topic configured in Chatbot channel
- Enable the rule
- Ensure SNS topic is in the Chatbot channel configuration

SHOULD:
- Filter findings by severity to reduce noise (add severity filter to event pattern)
- Test end-to-end with a sample finding
- Monitor rule invocation metrics

MAY:
- Create separate rules for different severity levels
- Route critical findings to a dedicated channel

## Common Issues

- symptoms: "Security Hub findings not appearing in Slack/Teams"
  diagnosis: "No EventBridge rule configured for Security Hub events."
  resolution: "Create an EventBridge rule matching Security Hub finding events targeting the Chatbot SNS topic."

- symptoms: "EventBridge rule exists but no notifications"
  diagnosis: "Rule target is not the SNS topic in the Chatbot channel, or rule is disabled."
  resolution: "Verify rule target and enable the rule."

- symptoms: "Too many Security Hub notifications flooding the channel"
  diagnosis: "EventBridge rule matches all findings without severity filtering."
  resolution: "Add severity filter to the event pattern (e.g., only CRITICAL and HIGH)."

## Output Format

```yaml
root_cause: "security_hub_integration_issue — <specific_cause>"
evidence:
  - type: eventbridge_rule
    content: "<rule pattern and targets>"
  - type: sns_config
    content: "<SNS topic and Chatbot channel>"
  - type: findings
    content: "<recent Security Hub findings>"
severity: MEDIUM
mitigation:
  immediate: "Create or fix EventBridge rule for Security Hub events"
  long_term: "Implement severity-based notification routing"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create EventBridge rule with Security Hub finding event pattern | GREEN |
| Set rule target to the SNS topic configured in Chatbot channel | GREEN |
| Enable the rule | GREEN |
| Ensure SNS topic is in the Chatbot channel configuration | GREEN |
| Filter findings by severity to reduce noise | GREEN |
| Test end-to-end with a sample finding | GREEN |
| Monitor rule invocation metrics | GREEN |
| Create separate rules for different severity levels | GREEN |
| Route critical findings to a dedicated channel | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- Security Hub findings may contain vulnerability details and security posture information

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If EventBridge rules were created, remove targets then delete rules
2. If severity filters were changed, restore the previous event pattern
3. If SNS topics were added to channel configuration, remove them if incorrect
4. Verify rollback by confirming EventBridge rules and Chatbot channel configuration match expected state

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
