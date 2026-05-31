---
title: "E1 — Execution Role Permission Issues"
description: "Diagnose Lambda execution role permission failures"
status: active
severity: HIGH
triggers:
  - "AccessDeniedException"
  - "is not authorized to perform"
  - "AccessDenied"
owner: devops-agent
objective: "Fix execution role permissions to allow function operations"
context: "The execution role is assumed by the Lambda service. It grants permissions for: CloudWatch Logs (required), VPC ENI management (if VPC), and any AWS services the function calls. Minimum: AWSLambdaBasicExecutionRole for logging."
---

## Common Issues

- symptoms: "AccessDeniedException calling S3/DynamoDB/SQS/etc."
  diagnosis: "Execution role missing permissions for the target service."
  resolution: "Add the required IAM permissions to the execution role."

- symptoms: "No CloudWatch logs appearing"
  diagnosis: "Execution role missing logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents."
  resolution: "Attach AWSLambdaBasicExecutionRole managed policy."

- symptoms: "VPC Lambda fails to create ENI"
  diagnosis: "Execution role missing ec2:CreateNetworkInterface, ec2:DescribeNetworkInterfaces, ec2:DeleteNetworkInterface."
  resolution: "Attach AWSLambdaVPCAccessExecutionRole managed policy."


## Safety Ratings

```
safety_ratings:
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
```

## Escalation Conditions

- Function serves a production API or critical workload
- Fix requires changing reserved concurrency settings
- Function is processing sensitive data (PII, financial, healthcare)
- Resolution involves modifying VPC configuration or security groups
- Multiple functions affected suggesting account-level issue

## Data Sensitivity

HIGH - Lambda function code and environment variables may contain secrets, API keys, database credentials, and encryption keys. CloudWatch logs may capture sensitive request/response data. X-Ray traces may contain PII in segment metadata.

## Prohibited Actions

- NEVER suggest setting reserved concurrency to 0 - this effectively disables the function
- NEVER suggest deleting a function alias that is serving live traffic
- NEVER recommend removing or replacing the execution role on a running function without verifying the new role has equivalent permissions
- NEVER suggest publishing function code changes directly to a production alias without testing
- NEVER expose environment variable values in logs or diagnostic output - they may contain secrets

## Phase 3 - Rollback

1. If VPC configuration was changed, restore original VPC settings: `aws lambda update-function-configuration --function-name <name> --vpc-config SubnetIds=<original>,SecurityGroupIds=<original>`
2. If execution role was modified, revert IAM policy changes and verify function can still access required resources
3. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<missing_service_permission|missing_logs_permission|missing_vpc_permission>"
evidence:
  - type: iam_error
    content: "<AccessDeniedException details>"
severity: HIGH
mitigation:
  immediate: "Add required permissions to execution role"
  long_term: "Use least-privilege policies, test with IAM Policy Simulator"
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
  - command: "get-function-configuration"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "invoke"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Resource: * in Lambda execution role"
  - "NEVER suggest disabling VPC configuration to fix connectivity"
  - "NEVER expose function URLs without authentication"
