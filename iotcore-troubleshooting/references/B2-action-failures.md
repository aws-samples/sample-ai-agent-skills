---
title: "B2 — Rule Action Failures"
description: "Diagnose IoT rule action execution failures"
status: active
severity: HIGH
triggers:
  - "action failed"
  - "rule action error"
  - "Lambda not invoked"
  - "DynamoDB write failed"
  - "S3 put failed"
owner: devops-agent
objective: "Identify and resolve rule action failures"
context: "Rule action failures occur when: IAM role lacks permissions for the target service, target resource doesn't exist, payload format incompatible, or target service is throttling."
---

## Phase 1 — Triage

MUST:
- Check rule actions configuration: `aws iot get-topic-rule --rule-name <rule-name> --query 'rule.actions'`
- Check IAM role policies: `aws iam list-role-policies --role-name <role-name>` and `aws iam list-attached-role-policies --role-name <role-name>`
- Check CloudWatch metrics for action failures: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name Failure --dimensions Name=RuleName,Value=<rule-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify target resource exists (Lambda function, DynamoDB table, S3 bucket, etc.)

SHOULD:
- Check error action output for specific error messages
- Verify IAM role trust policy allows iot.amazonaws.com
- Check target service CloudWatch logs for errors

MAY:
- Test action manually with IoT console
- Check target service quotas for throttling

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for the target service
- Ensure IAM role trust policy includes: `{"Effect":"Allow","Principal":{"Service":"iot.amazonaws.com"},"Action":"sts:AssumeRole"}`
- Verify target resource exists and is accessible

SHOULD:
- Add specific permissions for each action type (lambda:InvokeFunction, dynamodb:PutItem, s3:PutObject, etc.)
- Configure error action to capture failure details

MAY:
- Implement retry logic in error actions
- Set up monitoring for action success/failure rates

## Output Format

```yaml
root_cause: "action_failure — <specific_cause>"
evidence:
  - type: action_config
    content: "<rule action configuration>"
  - type: iam_role
    content: "<role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or target resource"
  long_term: "Implement error actions and action monitoring"
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
