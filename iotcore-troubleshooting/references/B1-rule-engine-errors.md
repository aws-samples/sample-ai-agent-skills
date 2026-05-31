---
title: "B1 — Rule Engine Errors"
description: "Diagnose IoT rule engine configuration and execution errors"
status: active
severity: HIGH
triggers:
  - "rule not triggering"
  - "rule engine error"
  - "topic rule failed"
  - "rule not working"
  - "messages not processed"
owner: devops-agent
objective: "Identify and resolve rule engine errors"
context: "Rule engine errors include: rule disabled, SQL syntax error, topic filter mismatch, IAM role permissions, or error action not configured."
---

## Phase 1 — Triage

MUST:
- Check rule configuration: `aws iot get-topic-rule --rule-name <rule-name>`
- Verify rule is enabled: check ruleDisabled field
- Check rule SQL syntax and topic filter
- Check IAM role for rule actions: `aws iam get-role --role-name <role-name>`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name RuleNotFound --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Test rule SQL in IoT console test feature
- Verify topic filter matches published topics
- Check error action configuration
- Review IoT logs for rule execution errors

MAY:
- Check for throttled rule executions
- Verify message payload matches SQL SELECT fields

## Phase 2 — Remediate

MUST:
- Enable disabled rule: `aws iot enable-topic-rule --rule-name <rule-name>`
- Fix SQL syntax errors in rule query
- Fix IAM role permissions for rule actions
- Configure error action for debugging

SHOULD:
- Add error action: `aws iot replace-topic-rule --rule-name <rule-name> --topic-rule-payload '{"sql":"...","actions":[...],"errorAction":{"cloudwatchLogs":{"logGroupName":"iot-rule-errors","roleArn":"<role-arn>"}}}'`

MAY:
- Implement rule versioning with CloudFormation
- Set up CloudWatch alarms on rule failures

## Output Format

```yaml
root_cause: "rule_engine_error — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule SQL and actions>"
  - type: rule_metrics
    content: "<execution and error counts>"
severity: HIGH
mitigation:
  immediate: "Fix rule SQL, enable rule, or fix IAM permissions"
  long_term: "Implement error actions and rule monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never revoke or delete device certificates without confirming device decommission
- Never modify IoT policies on production devices without testing in a staging environment
- Never delete thing groups that contain active production devices
- Never alter MQTT topic rules that route critical telemetry data
- Never disable custom authorizers serving active device connections

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reattach previous certificates if device connectivity is lost after changes
- Restore previous IoT policy versions if devices lose permissions
- Revert rule engine action configurations if message routing breaks

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
