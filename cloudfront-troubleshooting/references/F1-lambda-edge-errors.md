---
title: "F1 — Lambda@Edge Errors"
description: "Diagnose Lambda@Edge execution errors and configuration issues"
status: active
severity: CRITICAL
triggers:
  - "Lambda@Edge"
  - "edge function error"
  - "502 Lambda"
  - "503 Lambda"
  - "function execution"
owner: devops-agent
objective: "Resolve Lambda@Edge execution errors and configuration issues"
context: "Lambda@Edge runs Lambda functions at CloudFront edge locations. Functions MUST be in us-east-1. Viewer triggers: 5s timeout, 128 MB memory. Origin triggers: 30s timeout, up to 10 GB memory. No environment variables, no VPC, no layers, no ARM. Errors result in 502/503 to viewers. Logs go to CloudWatch in the edge region, not us-east-1."
---

## Phase 1 — Triage

MUST:
- Check function association: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.LambdaFunctionAssociations'`
- Check function configuration: `aws lambda get-function --function-name <function-name> --qualifier <version> --region us-east-1`
- Check CloudWatch logs in the EDGE REGION (not us-east-1): `aws logs describe-log-groups --log-group-name-prefix /aws/lambda/us-east-1.<function-name> --region <edge-region>`
- Check 5xx error rate in CloudWatch
- Verify the function version (not $LATEST — must be a published version)

SHOULD:
- Check function memory and timeout settings against limits
- Verify the function returns a valid CloudFront response object
- Check for unhandled exceptions in function code
- Verify IAM execution role has required permissions

MAY:
- Test the function locally with CloudFront event test payloads
- Check Lambda@Edge throttling metrics
- Review function code for common issues (async handling, response format)

## Phase 2 — Remediate

MUST:
- Deploy function in us-east-1 (required)
- Use a published version number (not $LATEST or alias)
- Return valid CloudFront response objects matching the expected schema
- Stay within limits: viewer 5s/128MB, origin 30s/configurable memory

SHOULD:
- Add error handling to return graceful error responses instead of crashing
- Minimize cold start impact by keeping functions small
- Use CloudFront Functions for simple viewer-level tasks instead

MAY:
- Implement canary deployments by gradually updating behaviors
- Set up CloudWatch alarms for Lambda@Edge errors across edge regions

## Common Issues

- symptoms: "502 LambdaExecutionError"
  diagnosis: "Function threw an unhandled exception or returned invalid response."
  resolution: "Check CloudWatch logs in the edge region. Fix error handling and response format."

- symptoms: "503 LambdaLimitExceeded"
  diagnosis: "Lambda@Edge concurrent execution limit reached."
  resolution: "Request limit increase. Optimize function to reduce execution time."

- symptoms: "Function works locally but fails at edge"
  diagnosis: "Using unsupported features: environment variables, VPC, layers, or ARM."
  resolution: "Remove unsupported features. Inline configuration values."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
```

## Escalation Conditions

- Distribution serves a production website or application
- Fix requires modifying origin configuration or cache behaviors
- Resolution involves certificate changes or HTTPS configuration
- Issue affects multiple distributions or is account-level
- Lambda@Edge or CloudFront Functions changes are required on production

## Data Sensitivity

MEDIUM - Signed URL private keys and key pairs control content access. Origin configurations may expose internal infrastructure (S3 bucket names, ALB endpoints). Access logs contain client IPs, request URIs, and query strings. Field-level encryption configurations protect sensitive form data.

## Prohibited Actions

- NEVER suggest deleting a CloudFront distribution that is serving live traffic
- NEVER suggest disabling HTTPS or downgrading the security policy on a production distribution
- NEVER recommend removing all cache behaviors - this breaks content routing
- NEVER suggest invalidating '/*' repeatedly as a fix - address the root caching issue instead
- NEVER recommend removing origin access control/identity from S3 origins without alternative access controls

## Phase 3 - Rollback

1. If distribution configuration was changed, update with previous settings: `aws cloudfront update-distribution --id <id> --distribution-config <previous> --if-match <etag>`
2. If cache policy or TTL was changed, restore original cache behavior settings and allow caches to repopulate
3. If edge function was changed, update the distribution to use the previous function version/ARN
4. If origin configuration was changed, restore original origin settings including timeouts and protocols
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "lambda_edge — <specific_cause>"
evidence:
  - type: function_config
    content: "<function ARN, version, memory, timeout>"
  - type: error_logs
    content: "<CloudWatch log entries from edge region>"
  - type: error_rate
    content: "<5xx error rate>"
severity: CRITICAL
mitigation:
  immediate: "Fix function error or remove association"
  long_term: "Implement error handling, monitoring across edge regions"
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
  - "NEVER suggest disabling HTTPS requirements"
  - "NEVER suggest removing WAF association to fix access"
  - "NEVER suggest wildcard CORS origins in production"
