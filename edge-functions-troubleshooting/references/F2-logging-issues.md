---
title: "F2 — Logging Issues"
description: "Diagnose edge function logging and debugging issues"
status: active
severity: MEDIUM
triggers: ["no logs", "logs missing", "cannot debug function", "log group not found", "logging not working"]
owner: devops-agent
objective: "Resolve edge function logging and observability issues"
context: "CF Functions log to /aws/cloudfront/function/<name>. Lambda@Edge logs to /aws/lambda/us-east-1.<name> in the EXECUTING region, not us-east-1."
---

## Phase 1 — Triage

MUST:
- For CF Functions, check log group exists: `aws logs describe-log-groups --log-group-name-prefix /aws/cloudfront/function/<function-name>`
- For Lambda@Edge, check logs in the EXECUTING region (not us-east-1): `aws logs describe-log-groups --log-group-name-prefix /aws/lambda/us-east-1.<function-name> --region <edge-region>`
- Verify the function includes console.log statements for debugging

SHOULD:
- Check if CloudFront real-time logs are configured: `aws cloudfront get-realtime-log-config --name <config-name>`
- Verify IAM permissions for CloudWatch Logs
- Check if the function's execution role has logs:CreateLogGroup and logs:PutLogEvents permissions

MAY:
- Enable CloudFront standard logging to S3
- Check multiple regions for Lambda@Edge logs (logs appear in the region where the function executed)

## Phase 2 — Remediate

MUST:
- For Lambda@Edge: search logs in ALL regions where the function may execute, not just us-east-1
- Ensure Lambda@Edge execution role has CloudWatch Logs permissions in all regions
- Add console.log statements to the function for debugging

SHOULD:
- Enable CloudFront real-time logs for detailed request/response data
- Create a log aggregation strategy for Lambda@Edge (logs are distributed across regions)
- Use structured logging (JSON) for easier parsing

MAY:
- Set up CloudWatch Logs Insights queries across regions
- Implement centralized logging with Kinesis Data Firehose
- Enable X-Ray tracing for Lambda@Edge

## Common Issues

- symptoms: "Lambda@Edge logs not found in us-east-1"
  diagnosis: "Lambda@Edge logs are created in the region where the function EXECUTES, not where it's deployed."
  resolution: "Check logs in the region closest to the user. Use: aws logs filter-log-events --region <edge-region>."

- symptoms: "CloudFront Function console.log output not appearing"
  diagnosis: "CF Function logs are only generated for a sample of requests, not all requests."
  resolution: "CF Functions log a sample of executions. Use test-function for guaranteed log output during development."

## Safety Ratings

```
safety_ratings:
  - "logs describe-log-groups: GREEN — read-only"
  - "logs filter-log-events: GREEN — read-only"
  - "cloudfront get-realtime-log-config: GREEN — read-only"
  - "cloudfront create-realtime-log-config: YELLOW — creates logging infrastructure"
```

## Escalation Conditions

- Logs are completely absent across all regions
- Log volume is too high and causing cost concerns
- Sensitive data is being logged inadvertently
- Log aggregation across regions is not feasible with current tools

## Data Sensitivity

- `aws logs filter-log-events`: HIGH — may contain request data, headers, and user information
- `aws logs describe-log-groups`: LOW — log group metadata only
- CloudFront real-time logs: HIGH — contain full request/response details
- Function console.log output: HIGH — developers may log sensitive request data

## Prohibited Actions

- NEVER log full request/response bodies in production
- NEVER log authentication tokens, cookies, or PII
- NEVER set log retention to unlimited — implement retention policies
- NEVER grant public access to CloudWatch log groups

## Phase 3 — Rollback

1. If logging changes caused performance issues, revert function code
2. If real-time log config was created incorrectly, delete it
3. If log retention was changed, restore previous setting
4. Verify logging works correctly after rollback
5. Document the logging architecture for the team

## Output Format

```yaml
root_cause: "logging — <specific_cause>"
evidence:
  - type: logging_issue
    content: "<log configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix log configuration or search correct regions"
  long_term: "Implement centralized log aggregation"
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
