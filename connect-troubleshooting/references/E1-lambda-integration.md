---
title: "E1 — Lambda Integration Failures"
description: "Diagnose Lambda function failures within Connect contact flows"
status: active
severity: HIGH
triggers:
  - "Lambda timeout in flow"
  - "Invoke Lambda failed"
  - "Lambda error in contact flow"
  - "8 second timeout"
owner: devops-agent
objective: "Fix Lambda integration issues in contact flows"
context: "Connect invokes Lambda with an 8-second hard timeout. The Lambda must return a flat JSON object for contact attributes. Common failures: timeout (function too slow), permission errors, incorrect response format, or function errors."
---

## Phase 1 — Triage

MUST:
- Check contact flow logs for Lambda invocation errors
- Verify the Lambda function is associated with the instance: `aws connect list-lambda-functions --instance-id <id>`
- Check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name>`
- Check Lambda function duration (must complete within 8 seconds)

SHOULD:
- Verify the Lambda function's IAM role has necessary permissions
- Check if the function is in a VPC (adds cold start latency)
- Test the Lambda function directly with a sample Connect event
- Check if the function returns the correct format: `{"key": "value"}`

MAY:
- Check Lambda concurrency and throttling metrics
- Review the function code for error handling
- Check if cold starts are causing timeouts

## Phase 2 — Remediate

MUST:
- Optimize the Lambda to complete within 8 seconds
- Fix the response format to return flat JSON: `{"key": "value"}`
- Associate the Lambda with the Connect instance if not already done
- Fix any permission errors

SHOULD:
- Add error handling in the Lambda function
- Use provisioned concurrency to eliminate cold starts
- Cache external service responses to reduce latency
- Implement the Error branch in the contact flow for Lambda failures

MAY:
- Use async patterns for operations that take longer than 8 seconds
- Implement connection pooling for database calls
- Move initialization code outside the handler

## Common Issues

- symptoms: "Lambda times out in contact flow but works when tested directly"
  diagnosis: "Lambda takes more than 8 seconds. Connect's limit is lower than the function's own timeout."
  resolution: "Optimize to complete within 8 seconds. Connect enforces 8s regardless of Lambda timeout setting."

- symptoms: "Lambda returns successfully but attributes not available in flow"
  diagnosis: "Lambda response format is incorrect (nested objects, arrays, or non-string values)."
  resolution: "Return flat JSON with string values: `{\"key\": \"value\"}`. No nested objects."

- symptoms: "Lambda invocation fails with permission error"
  diagnosis: "Lambda not associated with the Connect instance or resource policy missing."
  resolution: "Associate the Lambda via Connect console or API. Verify the resource policy allows Connect invocation."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If a contact flow was modified, publish the previous saved version of the flow from the Connect console
2. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
3. If agent routing or security profiles were modified, restore original profile assignments
4. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "lambda_integration — <specific cause>"
evidence:
  - type: lambda_logs
    content: "<function error or duration>"
  - type: contact_flow_logs
    content: "<invocation error>"
severity: HIGH
mitigation:
  immediate: "Fix the Lambda function or configuration"
  long_term: "Optimize Lambda performance and implement error handling"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
