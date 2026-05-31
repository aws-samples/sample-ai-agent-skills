---
title: "E3 — Custom Authorizer Issues"
description: "Diagnose custom authorizer configuration and execution problems"
status: active
severity: HIGH
triggers:
  - "custom authorizer failed"
  - "authorizer timeout"
  - "token validation error"
  - "authorizer not invoked"
  - "Lambda authorizer error"
owner: devops-agent
objective: "Identify and resolve custom authorizer issues"
context: "Custom authorizer issues include: Lambda function errors, timeout (5s limit), invalid response format, token signing misconfigured, or authorizer not set as default."
---

## Phase 1 — Triage

MUST:
- Check authorizer configuration: `aws iot describe-authorizer --authorizer-name <name>`
- Check default authorizer: `aws iot describe-default-authorizer`
- Check Lambda function: `aws lambda get-function --function-name <function-name>`
- Check Lambda logs for errors: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name>`

SHOULD:
- Verify Lambda execution role has required permissions
- Check Lambda timeout (must be under 5 seconds)
- Verify response format includes isAuthenticated, principalId, policyDocuments
- Check token signing configuration if enabled

MAY:
- Test authorizer: `aws iot test-invoke-authorizer --authorizer-name <name> --token <token>`
- Check authorizer caching settings

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors
- Ensure Lambda timeout is under 5 seconds
- Return valid response format from Lambda
- Set authorizer as default if needed: `aws iot set-default-authorizer --authorizer-name <name>`

SHOULD:
- Enable caching to reduce Lambda invocations
- Implement proper error handling in Lambda
- Configure token signing for security

MAY:
- Implement authorizer monitoring and alerting
- Use caching with appropriate TTL

## Output Format

```yaml
root_cause: "authorizer_issue — <specific_cause>"
evidence:
  - type: authorizer_config
    content: "<authorizer configuration>"
  - type: lambda_logs
    content: "<Lambda execution errors>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda function or authorizer configuration"
  long_term: "Implement robust authorizer with caching and monitoring"
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
