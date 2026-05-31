---
title: "G2 — S3/DynamoDB/Kinesis Action Issues"
description: "Diagnose IoT rule actions for S3, DynamoDB, and Kinesis"
status: active
severity: HIGH
triggers:
  - "S3 action failed"
  - "DynamoDB put failed"
  - "Kinesis write failed"
  - "action permission error"
  - "data not arriving"
owner: devops-agent
objective: "Identify and resolve S3, DynamoDB, and Kinesis action issues"
context: "These action failures typically involve: IAM role missing service permissions, target resource not found, payload format issues, or target service throttling."
---

## Phase 1 — Triage

MUST:
- Check rule actions: `aws iot get-topic-rule --rule-name <rule-name> --query 'rule.actions'`
- Check IAM role and policies: `aws iam get-role --role-name <role-name>` and `aws iam list-attached-role-policies --role-name <role-name>`
- Verify target resource exists (S3 bucket, DynamoDB table, Kinesis stream)
- Check rule error metrics and error action output

SHOULD:
- Verify IAM role trust policy allows iot.amazonaws.com
- Check target service for throttling or capacity issues
- Verify DynamoDB table key schema matches action configuration
- Check Kinesis stream shard count for throughput

MAY:
- Check S3 bucket policy for additional restrictions
- Verify DynamoDB provisioned capacity
- Check Kinesis stream retention period

## Phase 2 — Remediate

MUST:
- Fix IAM role with required permissions:
  - S3: s3:PutObject on the bucket ARN
  - DynamoDB: dynamodb:PutItem on the table ARN
  - Kinesis: kinesis:PutRecord on the stream ARN
- Ensure trust policy: `{"Effect":"Allow","Principal":{"Service":"iot.amazonaws.com"},"Action":"sts:AssumeRole"}`

SHOULD:
- Configure error action for debugging failures
- Match DynamoDB action hashKeyValue/rangeKeyValue to table schema
- Set appropriate Kinesis partition key

MAY:
- Implement DynamoDB auto-scaling for variable throughput
- Use Kinesis enhanced fan-out for high-throughput consumers

## Output Format

```yaml
root_cause: "service_action_issue — <specific_cause>"
evidence:
  - type: action_config
    content: "<action configuration>"
  - type: iam_permissions
    content: "<role permissions>"
  - type: target_resource
    content: "<target resource status>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or target resource configuration"
  long_term: "Implement comprehensive action monitoring with error actions"
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
