---
title: "G1 — Lambda Action Issues"
description: "Diagnose IoT rule Lambda action integration problems"
status: active
severity: HIGH
triggers:
  - "Lambda action failed"
  - "Lambda not invoked"
  - "Lambda timeout"
  - "Lambda permission error"
  - "function error"
owner: devops-agent
objective: "Identify and resolve Lambda action integration issues"
context: "Lambda action issues include: missing invoke permissions, function errors, timeout, payload format issues, or function not found."
---

## Phase 1 — Triage

MUST:
- Check rule Lambda action config: `aws iot get-topic-rule --rule-name <rule-name> --query 'rule.actions[?lambda]'`
- Check Lambda function exists: `aws lambda get-function --function-name <function-name>`
- Check Lambda resource policy allows IoT: `aws lambda get-policy --function-name <function-name>`
- Check Lambda invocation errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check Lambda CloudWatch logs for execution errors
- Verify Lambda timeout is sufficient
- Check Lambda memory allocation
- Verify payload format matches function expectations

MAY:
- Test Lambda with sample IoT payload
- Check Lambda concurrency limits

## Phase 2 — Remediate

MUST:
- Add IoT permission to Lambda: `aws lambda add-permission --function-name <function> --statement-id iot-rule --action lambda:InvokeFunction --principal iot.amazonaws.com --source-arn arn:aws:iot:<region>:<account>:rule/<rule-name>`
- Fix Lambda function errors
- Ensure Lambda timeout is appropriate

SHOULD:
- Implement error handling in Lambda function
- Configure error action on the rule for failed invocations
- Set up Lambda dead-letter queue

MAY:
- Implement Lambda retry logic
- Monitor Lambda invocation metrics

## Output Format

```yaml
root_cause: "lambda_action_issue — <specific_cause>"
evidence:
  - type: lambda_config
    content: "<Lambda function configuration>"
  - type: invocation_errors
    content: "<Lambda error logs>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda permissions or function errors"
  long_term: "Implement robust error handling and monitoring"
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
