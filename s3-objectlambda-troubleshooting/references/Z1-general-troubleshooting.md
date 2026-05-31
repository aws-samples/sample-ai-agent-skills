---
title: "Z1 — General Troubleshooting"
description: "General S3 Object Lambda troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Object Lambda error", "OLAP not working", "general OLAP issue", "unknown Object Lambda problem"]
owner: devops-agent
objective: "Systematically diagnose and resolve general S3 Object Lambda issues"
context: "Catch-all runbook for Object Lambda issues that don't fit specific categories. Follows a systematic elimination approach."
---

## Phase 1 — Triage

MUST:
- List all OLAPs in the account: `aws s3control list-access-points-for-object-lambda --account-id <account-id>`
- Check OLAP status and configuration: `aws s3control get-access-point-for-object-lambda --account-id <account-id> --name <olap-name>`
- Check Lambda function health: `aws lambda get-function --function-name <function-name>`

SHOULD:
- Review CloudWatch metrics for errors: `aws cloudwatch get-metric-statistics --namespace AWS/S3ObjectLambda --metric-name 4xxErrors --dimensions Name=AccessPointName,Value=<olap-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CloudTrail for recent API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=s3-object-lambda.amazonaws.com --max-results 20`
- Verify supporting access point exists and is healthy

MAY:
- Check AWS Health Dashboard for S3 Object Lambda service issues
- Review recent changes in CloudTrail that may have caused the issue
- Test with a minimal Lambda function to isolate the problem

## Phase 2 — Remediate

MUST:
- Address the specific error identified in triage
- Verify all three policy layers (OLAP, supporting AP, IAM) are correctly configured
- Ensure Lambda function calls WriteGetObjectResponse

SHOULD:
- Test the fix with a non-production OLAP first
- Monitor CloudWatch metrics after applying the fix
- Document the root cause and resolution

MAY:
- Set up CloudWatch alarms for OLAP error rates
- Implement automated health checks for OLAPs
- Create a runbook for the specific issue if it's recurring

## Common Issues

- symptoms: "All OLAP requests return 500 Internal Server Error"
  diagnosis: "Lambda function is not calling WriteGetObjectResponse, or function is crashing."
  resolution: "Check Lambda logs. Ensure WriteGetObjectResponse is called in all code paths including error handlers."

- symptoms: "OLAP works intermittently — some requests succeed, others fail"
  diagnosis: "Lambda throttling, cold start timeouts, or transient S3 errors."
  resolution: "Check throttle metrics. Increase concurrency. Add retry logic in the Lambda function."

## Safety Ratings

```
safety_ratings:
  - "list-access-points-for-object-lambda: GREEN — read-only enumeration"
  - "get-access-point-for-object-lambda: GREEN — read-only"
  - "lambda get-function: GREEN — read-only"
  - "cloudtrail lookup-events: GREEN — read-only audit log search"
```

## Escalation Conditions

- Issue cannot be diagnosed with available logs and metrics
- Problem appears to be an AWS service issue (consistent 500s with correct configuration)
- Multiple OLAPs affected simultaneously suggesting account-level or regional issue
- Issue requires AWS Support engagement for internal service debugging

## Data Sensitivity

- `aws s3control list-access-points-for-object-lambda`: LOW — OLAP metadata only
- `aws lambda get-function`: MEDIUM — may reveal environment variables
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principals
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics

## Prohibited Actions

- NEVER delete OLAPs or supporting access points as a troubleshooting step
- NEVER modify production Lambda functions without a rollback plan
- NEVER assume the issue is with AWS service — verify configuration first
- NEVER share CloudTrail logs containing principal ARNs externally without redaction

## Phase 3 — Rollback

1. If any configuration changes were made during troubleshooting, revert them
2. If Lambda function was modified, roll back to the last known working version
3. If policies were changed, restore previous versions
4. Verify the system returns to its pre-troubleshooting state
5. Document all changes made and their outcomes

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: diagnostic_finding
    content: "<finding details>"
severity: MEDIUM
mitigation:
  immediate: "Apply targeted fix"
  long_term: "Implement monitoring and alerting for early detection"
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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"
