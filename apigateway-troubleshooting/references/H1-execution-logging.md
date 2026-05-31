---
title: "H1 — Execution Logging Issues"
description: "Diagnose API Gateway execution logging configuration and missing log entries"
status: active
severity: MEDIUM
triggers:
  - "execution log"
  - "CloudWatch logs"
  - "logging"
  - "no logs"
  - "missing logs"
  - "log group"
owner: devops-agent
objective: "Resolve execution logging issues by configuring CloudWatch log groups, IAM roles, and log levels"
context: "API Gateway execution logs provide detailed request/response information including integration request/response, authorizer output, and error details. Execution logging requires a CloudWatch Logs role ARN set at the account level and logging enabled per stage/method. Log groups follow the naming pattern: API-Gateway-Execution-Logs_<api-id>/<stage>. Common issues include missing IAM role, insufficient permissions, and log level configuration."
---

## Phase 1 — Triage

MUST:
- Check account-level CloudWatch role: `aws apigateway get-account --query 'cloudwatchRoleArn'`
- Check stage logging settings: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'methodSettings."*/*".{loggingLevel:loggingLevel,dataTraceEnabled:dataTraceEnabled}'`
- Check if log group exists: `aws logs describe-log-groups --log-group-name-prefix API-Gateway-Execution-Logs_<api-id>`
- Verify IAM role has CloudWatch Logs permissions: `aws iam get-role-policy --role-name <role-name> --policy-name <policy-name>`
- Check for recent log entries: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --start-time <epoch-ms> --limit 10`

SHOULD:
- Verify the CloudWatch role trust policy allows apigateway.amazonaws.com
- Check if logging is enabled at method level (overrides stage default)
- Verify log group retention policy

MAY:
- Check CloudTrail for logging configuration changes
- Verify no resource policy on the log group blocking writes

## Phase 2 — Remediate

MUST:
- Set account-level CloudWatch role: `aws apigateway update-account --patch-operations op=replace,path=/cloudwatchRoleArn,value=<role-arn>`
- Ensure IAM role has policy: `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:DescribeLogGroups`, `logs:DescribeLogStreams`, `logs:PutLogEvents`, `logs:GetLogGroupFields`, `logs:GetLogEvents`, `logs:FilterLogEvents`
- Enable execution logging: `aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> --patch-operations op=replace,path=/*/*/logging/loglevel,value=INFO`

SHOULD:
- Enable data trace for debugging (includes full request/response bodies): `aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> --patch-operations op=replace,path=/*/*/logging/dataTrace,value=true`
- Set appropriate log retention to manage costs
- Disable data trace in production (contains sensitive data)

MAY:
- Use ERROR log level in production, INFO for debugging
- Set up log-based CloudWatch alarms for error patterns

## Common Issues

- symptoms: "No execution logs appearing in CloudWatch"
  diagnosis: "CloudWatch role ARN not set at account level or role lacks permissions."
  resolution: "Set account CloudWatch role with proper permissions for apigateway.amazonaws.com."

- symptoms: "Log group exists but no new entries"
  diagnosis: "Logging level set to OFF at stage or method level."
  resolution: "Set logging level to INFO or ERROR in stage method settings."

- symptoms: "Logs appear but missing request/response bodies"
  diagnosis: "Data trace is disabled (default). Only metadata is logged."
  resolution: "Enable data trace for full request/response logging. Disable in production."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- API serves production traffic from external or internal consumers
- Fix requires redeploying a production stage
- Resolution involves modifying authorizer configuration
- Issue affects multiple APIs or is account-level throttling
- Custom domain name or certificate changes are required

## Data Sensitivity

MEDIUM - API keys and usage plan configurations control access to APIs. Authorizer configurations (Lambda authorizer code, Cognito pool settings) are security-sensitive. Access logs may contain request parameters, headers, and client IPs. Mapping templates may expose internal backend URLs and data transformation logic.

## Prohibited Actions

- NEVER suggest deleting a production stage - this immediately removes the API endpoint
- NEVER suggest removing all authorizers from an API that requires authentication
- NEVER recommend disabling API key requirements on usage-plan-protected endpoints
- NEVER suggest modifying throttling limits to unlimited on a production API
- NEVER recommend deleting and recreating a custom domain name - this causes DNS propagation downtime

## Phase 3 - Rollback

1. If a deployment was made, redeploy the previous working deployment to the stage: `aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>`
2. If authorizer configuration was changed, restore the original authorizer settings and redeploy the stage
3. If integration settings were changed, restore original integration configuration and redeploy
4. If CORS or method configuration was changed, restore original settings and redeploy the API
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "execution_logging — <specific_cause>"
evidence:
  - type: account_role
    content: "<CloudWatch role ARN and permissions>"
  - type: stage_settings
    content: "<logging level and data trace settings>"
  - type: log_group
    content: "<log group existence and recent entries>"
severity: MEDIUM
mitigation:
  immediate: "Configure CloudWatch role and enable logging"
  long_term: "Implement logging strategy with appropriate levels per environment"
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
  - "NEVER suggest removing authorizers to fix access"
  - "NEVER suggest disabling throttling"
  - "NEVER suggest open CORS for production APIs"
