---
title: "D1 — Access Logging"
description: "Diagnose MediaStore access logging configuration issues"
status: active
severity: MEDIUM
triggers: ["access logging not working", "no logs", "logging configuration", "cloudwatch logs missing"]
owner: devops-agent
objective: "Resolve MediaStore access logging configuration and delivery issues"
context: "MediaStore access logs are delivered to CloudWatch Logs, not S3. Logging must be explicitly enabled per container."
---

## Phase 1 — Triage

MUST:
- Check logging status: `aws mediastore describe-container --container-name <name> --query 'Container.AccessLoggingEnabled'`
- Verify CloudWatch log group exists: `aws logs describe-log-groups --log-group-name-prefix /aws/mediastore/<container-name>`
- Check IAM permissions for logs:CreateLogGroup and logs:PutLogEvents

SHOULD:
- Check for recent log events: `aws logs get-log-events --log-group-name /aws/mediastore/<container-name> --log-stream-name <stream> --limit 10`
- Verify the container has recent activity that should generate logs

MAY:
- Check CloudWatch Logs service quotas
- Review log retention settings

## Phase 2 — Remediate

MUST:
- Enable access logging: `aws mediastore start-access-logging --container-name <name>`
- Verify logging is enabled: `aws mediastore describe-container --container-name <name>`
- Ensure the MediaStore service has permission to write to CloudWatch Logs

SHOULD:
- Set log retention to control costs: `aws logs put-retention-policy --log-group-name /aws/mediastore/<container-name> --retention-in-days 30`
- Create CloudWatch alarms based on access log patterns

MAY:
- Export logs to S3 for long-term analysis

## Safety Ratings

```
safety_ratings:
  - "mediastore describe-container: GREEN — read-only container query"
  - "mediastore start-access-logging: GREEN — enables logging, non-destructive"
  - "mediastore stop-access-logging: YELLOW — stops logging, may lose visibility"
  - "logs put-retention-policy: YELLOW — may cause old logs to be deleted"
```

## Escalation Conditions

- Logging is enabled but no log events appear after 15 minutes of activity
- CloudWatch Logs service quota prevents log delivery
- Log volume is excessive and causing unexpected CloudWatch costs
- Need to correlate access logs with CloudFront logs for end-to-end tracing

## Data Sensitivity

- Access logs: HIGH — contain requester IPs, paths, and response codes
- Log group configuration: LOW — retention and subscription settings
- Container logging status: LOW — enabled/disabled flag
- Log stream names: LOW — identifiers

## Prohibited Actions

- NEVER disable access logging on production containers without approval
- NEVER set log retention to 0 (never expire) without understanding cost implications
- NEVER share access logs containing requester IP addresses publicly
- NEVER delete CloudWatch log groups with active logging — logs will be lost

## Phase 3 — Rollback

1. If logging was disabled, re-enable: `aws mediastore start-access-logging --container-name <name>`
2. If retention was changed, revert: `aws logs put-retention-policy --log-group-name <name> --retention-in-days <original>`
3. If log group was deleted, it will be recreated when logging generates new events
4. Verify logs are flowing after restoration
5. Document the logging configuration change

## Common Issues

- symptoms: "Access logging is enabled but CloudWatch log group is empty"
  diagnosis: "No requests have been made to the container, or there is a delay in log delivery."
  resolution: "Make a test request to the container. Wait up to 5 minutes for logs to appear."

- symptoms: "CloudWatch Logs costs are unexpectedly high for MediaStore"
  diagnosis: "High-traffic containers generate large volumes of access logs."
  resolution: "Set a retention policy to limit log storage. Consider sampling or filtering logs."

## Output Format

```yaml
root_cause: "access_logging — <specific_cause>"
evidence:
  - type: logging_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Enable or fix logging configuration"
  long_term: "Set retention policies and cost alerts"
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
