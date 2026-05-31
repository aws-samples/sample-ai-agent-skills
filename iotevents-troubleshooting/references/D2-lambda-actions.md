---
title: "D2 — Lambda Action Issues"
description: "Diagnose Lambda function action failures"
status: active
severity: MEDIUM
triggers:
  - "Lambda not invoked"
  - "Lambda timeout"
  - "Lambda error in action"
owner: devops-agent
objective: "Identify and resolve Lambda action failures"
context: "Lambda action failures occur when: function doesn't exist, IAM role lacks lambda:InvokeFunction, function errors, or timeout."
---

## Phase 1 — Triage

MUST:
- Verify Lambda function exists: `aws lambda get-function --function-name <name>`
- Check IAM role has lambda:InvokeFunction permission
- Check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch>`
- Verify function ARN in action configuration

SHOULD:
- Check Lambda function timeout vs expected execution time
- Verify payload format matches function expectations
- Check Lambda concurrent execution limits

MAY:
- Test Lambda function manually: `aws lambda invoke --function-name <name> --payload '<json>' output.json`
- Check Lambda DLQ for failed invocations

## Phase 2 — Remediate

MUST:
- Grant lambda:InvokeFunction to IoT Events IAM role
- Fix Lambda function ARN in action
- Fix Lambda function errors

SHOULD:
- Configure Lambda DLQ for failed invocations
- Set appropriate Lambda timeout

## Output Format

```yaml
root_cause: "lambda_action — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<function error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions or function errors"
  long_term: "Implement Lambda error handling and DLQ"
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
- Never delete detector models that are actively monitoring production inputs
- Never modify alarm models during active alarm states without understanding impact
- Never change input routing configurations without verifying downstream detectors
- Never alter state transition logic on production detector models without testing
- Never remove action targets (Lambda, SNS, etc.) from active detector models

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Republish previous detector model version if event processing breaks
- Restore previous alarm model configuration if alarm evaluation fails
- Revert input routing changes if detectors stop receiving data

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
