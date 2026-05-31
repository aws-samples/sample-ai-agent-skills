---
title: "H2 — S3 Lambda Trigger Issues"
description: "Diagnose S3 event-triggered Lambda function failures"
status: active
severity: HIGH
triggers:
  - "Lambda trigger not working"
  - "S3 Lambda invocation failed"
  - "Recursive Lambda invocation"
  - "Lambda resource policy"
owner: devops-agent
objective: "Identify and fix S3-to-Lambda trigger configuration and execution issues"
context: "S3 can invoke Lambda functions on object events. The Lambda function needs a resource policy allowing s3.amazonaws.com. The event payload includes bucket name, key, size, and ETag. Recursive invocations (Lambda writes to the same bucket that triggers it) can cause infinite loops and high costs."
---

## Phase 1 — Triage

MUST:
- Check Lambda resource policy: `aws lambda get-policy --function-name <function-name>`
- Verify S3 notification configuration includes the Lambda ARN: `aws s3api get-bucket-notification-configuration --bucket <bucket>`
- Check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Check for recursive invocation risk (Lambda writes to the triggering bucket)

SHOULD:
- Verify the Lambda execution role has permissions for its operations
- Check Lambda concurrency limits and throttling
- Verify the event format matches what the function expects

MAY:
- Check Lambda dead-letter queue for failed invocations
- Monitor Lambda concurrent executions metric

## Phase 2 — Remediate

MUST:
- Add Lambda resource policy if missing: `aws lambda add-permission --function-name <fn> --statement-id s3-invoke --action lambda:InvokeFunction --principal s3.amazonaws.com --source-arn arn:aws:s3:::<bucket> --source-account <account-id>`
- Fix recursive invocations: use a different output bucket or prefix, or add S3 event prefix filters
- Ensure the Lambda function handles the S3 event format correctly

SHOULD:
- Add error handling for missing keys or access denied in the Lambda function
- Set up a dead-letter queue for failed invocations
- Use reserved concurrency to limit blast radius

MAY:
- Use Lambda Destinations for async invocation result routing
- Implement idempotency in the Lambda function (S3 may deliver events more than once)

## Common Issues

- symptoms: "Lambda not invoked when objects are uploaded"
  diagnosis: "Lambda resource policy does not allow s3.amazonaws.com."
  resolution: "Add the resource policy with lambda:InvokeFunction for s3.amazonaws.com."

- symptoms: "Lambda invoked thousands of times unexpectedly"
  diagnosis: "Recursive invocation: Lambda writes to the same bucket/prefix that triggers it."
  resolution: "Use a different output bucket or prefix. Add prefix filters to the notification."

- symptoms: "Lambda fails with AccessDenied reading the S3 object"
  diagnosis: "Lambda execution role lacks s3:GetObject permission."
  resolution: "Add s3:GetObject to the Lambda execution role for the bucket ARN."

## Output Format

```yaml
root_cause: "lambda_trigger — <specific_cause>"
evidence:
  - type: resource_policy
    content: "<Lambda resource policy>"
  - type: lambda_logs
    content: "<error log entries>"
severity: HIGH
mitigation:
  immediate: "Fix resource policy or recursive invocation"
  long_term: "Add dead-letter queue, concurrency limits, and idempotency"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying Lambda resource policies and S3 notification configuration via lambda add-permission and put-bucket-notification-configuration. Recursive invocation fixes are critical. Uses get-policy and get-bucket-notification-configuration for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Lambda trigger changes affect event-driven production workflows

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Lambda trigger configuration reveals integration architecture"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
