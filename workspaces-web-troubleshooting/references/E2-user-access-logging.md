---
title: "E2 — User Access Logging"
description: "Diagnose WorkSpaces Web user access logging issues"
status: active
severity: MEDIUM
triggers: ["logging not working", "no session logs", "Kinesis stream empty", "audit logs missing"]
owner: devops-agent
objective: "Resolve user access logging configuration issues for audit and compliance"
context: "User access logging sends session activity data to an Amazon Kinesis Data Stream. The Kinesis stream must exist and the WorkSpaces Web service must have permission to write to it. Logging is not enabled by default."
---

## Phase 1 — Triage

MUST:
- Check logging settings: `aws workspaces-web get-user-access-logging-settings --user-access-logging-settings-arn <arn>`
- Verify the Kinesis Data Stream exists and is ACTIVE: `aws kinesis describe-stream --stream-name <stream-name> --query 'StreamDescription.{StreamStatus:StreamStatus,StreamARN:StreamARN}'`
- Check if logging settings are associated with the portal: `aws workspaces-web get-portal --portal-arn <portal-arn>`

SHOULD:
- Verify IAM permissions for the WorkSpaces Web service to write to Kinesis
- Check Kinesis stream metrics for incoming records: `aws cloudwatch get-metric-statistics --namespace AWS/Kinesis --metric-name IncomingRecords --dimensions Name=StreamName,Value=<stream-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Check Kinesis stream consumer configuration if logs are not being processed
- Review CloudTrail for logging settings changes

## Phase 2 — Remediate

MUST:
- Create Kinesis Data Stream if it does not exist: `aws kinesis create-stream --stream-name workspaces-web-access-logs --shard-count 1`
- Create user access logging settings: `aws workspaces-web create-user-access-logging-settings --kinesis-stream-arn <kinesis-stream-arn>`
- Associate logging settings with the portal: `aws workspaces-web associate-user-access-logging-settings --portal-arn <portal-arn> --user-access-logging-settings-arn <arn>`

SHOULD:
- Ensure the Kinesis stream has sufficient shard capacity for the expected log volume
- Configure a Kinesis consumer (Lambda, Firehose) to process and store logs
- Set up CloudWatch alarms for Kinesis stream errors

MAY:
- Configure Kinesis Data Firehose for automatic S3 archival of logs
- Set up log analysis with Amazon OpenSearch or Athena
- Implement log retention policies

## Safety Ratings

```
safety_ratings:
  - "get-user-access-logging-settings: GREEN — read-only logging configuration"
  - "describe-stream: GREEN — read-only Kinesis stream status"
  - "create-stream: YELLOW — creates billable Kinesis resource"
  - "create-user-access-logging-settings: YELLOW — creates logging configuration"
  - "associate-user-access-logging-settings: YELLOW — enables logging for portal sessions"
```

## Escalation Conditions

- Kinesis stream is throttling due to insufficient shard capacity
- Logging settings association fails with permission errors
- Compliance requires logging but Kinesis costs are prohibitive
- Log data contains unexpected or missing fields

## Data Sensitivity

- `aws workspaces-web get-user-access-logging-settings`: LOW — returns logging configuration
- `aws kinesis describe-stream`: LOW — returns stream metadata
- User access log records: HIGH — contain user session activity, URLs visited, and timestamps
- Kinesis stream data: HIGH — contains sensitive user activity data

## Prohibited Actions

- NEVER access user session logs without proper authorization and audit trail
- NEVER delete the Kinesis stream without archiving existing log data
- NEVER share user access logs outside of authorized security and compliance teams
- NEVER disable logging on portals subject to compliance requirements

## Phase 3 — Rollback

1. Disassociate logging settings if they cause portal issues: `aws workspaces-web disassociate-user-access-logging-settings --portal-arn <portal-arn>`
2. Delete incorrectly created logging settings: `aws workspaces-web delete-user-access-logging-settings --user-access-logging-settings-arn <arn>`
3. Delete Kinesis stream if created in error (after archiving any data): `aws kinesis delete-stream --stream-name <stream-name>`
4. Verify portal sessions are not affected by logging changes
5. Document the logging configuration change and reason for rollback

## Common Issues

- symptoms: "Kinesis stream exists but no records are arriving"
  diagnosis: "User access logging settings are not associated with the portal."
  resolution: "Associate the logging settings: aws workspaces-web associate-user-access-logging-settings."

## Output Format

```yaml
root_cause: "user_access_logging — <specific_cause>"
evidence:
  - type: logging_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix logging configuration"
  long_term: "Implement log processing pipeline and retention policies"
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
