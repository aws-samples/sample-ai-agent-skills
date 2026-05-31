---
title: "F1 — Custom Connector Issues"
description: "Diagnose Amazon AppFlow custom connector Lambda-based issues"
status: active
severity: HIGH
triggers: ["custom connector error", "Lambda connector failed", "custom connector SDK error"]
owner: devops-agent
objective: "Resolve custom connector issues including Lambda function errors, SDK response format, and authentication"
context: "Custom connectors use Lambda functions implementing the AppFlow Custom Connector SDK. Must return responses in exact SDK format."
---

## Phase 1 — Triage

MUST:
- List custom connectors: `aws appflow list-connectors --query 'connectors[?connectorType==CUSTOMCONNECTOR]'`
- Check Lambda function status: `aws lambda get-function --function-name <function-name>`
- Review Lambda invocation errors: `aws lambda get-function --function-name <function-name> --query 'Configuration.{State:State,LastModified:LastModified}'`

SHOULD:
- Check Lambda CloudWatch Logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --limit 50`
- Verify Lambda execution role has required permissions

MAY:
- Test Lambda function independently with a sample AppFlow SDK request
- Review Lambda memory and timeout configuration

## Phase 2 — Remediate

MUST:
- Fix Lambda function to return responses in exact AppFlow Custom Connector SDK format
- Ensure Lambda execution role has permissions for target API/service
- Fix Lambda timeout (must complete within AppFlow's invocation timeout)

SHOULD:
- Add structured error handling in Lambda function
- Implement pagination correctly per SDK specification

MAY:
- Add Lambda layers for shared connector SDK dependencies
- Implement connector caching for authentication tokens

## Safety Ratings

```
safety_ratings:
  - "list-connectors: GREEN — read-only connector enumeration"
  - "lambda get-function: GREEN — read-only function details"
  - "logs filter-log-events: GREEN — read-only log review"
  - "lambda update-function-code: RED — modifies Lambda function code"
```

## Escalation Conditions

- Custom connector SDK format is unclear for the specific use case
- Lambda function requires VPC access to reach the target API
- Custom connector authentication mechanism is complex (mTLS, custom OAuth)
- Lambda concurrency limits affect flow execution throughput

## Data Sensitivity

- `aws lambda get-function`: LOW — reveals function configuration
- `aws logs filter-log-events`: HIGH — may contain API responses and data samples
- Lambda invocation payloads: HIGH — contain source data being transferred

## Prohibited Actions

- NEVER deploy untested Lambda code to a production custom connector
- NEVER log sensitive data (API keys, tokens, PII) in Lambda function
- NEVER increase Lambda timeout beyond 15 minutes (Lambda maximum)
- NEVER bypass SDK response format validation — AppFlow will reject malformed responses

## Phase 3 — Rollback

1. If Lambda code was updated, revert to previous version: `aws lambda update-function-code --function-name <name> --s3-bucket <bucket> --s3-key <previous-version>`
2. If Lambda configuration was changed, restore previous settings
3. Verify custom connector still works with existing flows
4. Check Lambda CloudWatch Logs for errors after rollback
5. Document the Lambda version and configuration for the working state

## Common Issues

- symptoms: "Custom connector returns 'InvalidConnectorRuntimeSettingException'"
  diagnosis: "Lambda response does not match AppFlow Custom Connector SDK format."
  resolution: "Review SDK documentation. Ensure response includes all required fields: isSuccess, errorDetails, records with proper schema."

## Output Format

```yaml
root_cause: "custom_connector — <specific_cause>"
evidence:
  - type: lambda_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda function"
  long_term: "Add SDK response validation"
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
