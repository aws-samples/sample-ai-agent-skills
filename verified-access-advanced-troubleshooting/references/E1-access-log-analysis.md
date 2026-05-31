---
title: "E1 — Access Log Analysis"
description: "Analyze Verified Access access logs for troubleshooting"
status: active
severity: MEDIUM
triggers: ["access log analysis", "why was access denied", "log not showing", "logging not configured", "audit trail"]
owner: devops-agent
objective: "Configure and analyze Verified Access access logs for troubleshooting and auditing"
context: "Access logs contain trust context, policy evaluation results, and denial reasons. Logs can be sent to CloudWatch Logs, S3, or Kinesis Data Firehose. Logging must be explicitly enabled."
---

## Phase 1 — Triage

MUST:
- Check logging configuration: `aws ec2 describe-verified-access-instance-logging-configurations --verified-access-instance-ids <instance-id>`
- Query access logs for denials: `aws logs filter-log-events --log-group-name <va-log-group> --filter-pattern '{ $.activity = "denied" }' --start-time <epoch-ms>`
- Verify log group exists and has recent data: `aws logs describe-log-groups --log-group-name-prefix <va-log-prefix>`

SHOULD:
- Check log retention settings
- Verify IAM permissions for log delivery
- Review log format and available fields

MAY:
- Set up CloudWatch Insights queries for common patterns
- Configure log-based alarms for access denial spikes

## Phase 2 — Remediate

MUST:
- Enable logging if not configured: `aws ec2 modify-verified-access-instance-logging-configuration --verified-access-instance-id <id> --access-logs CloudWatchLogs={Enabled=true,LogGroup=<log-group>}`
- Ensure IAM role allows log delivery to the destination
- Verify logs are being written after enabling

SHOULD:
- Set appropriate log retention periods
- Configure log-based metrics for monitoring

MAY:
- Set up CloudWatch dashboards for access patterns
- Implement automated log analysis for security events

## Common Issues

- symptoms: "No access logs available for troubleshooting"
  diagnosis: "Logging not enabled on the Verified Access instance."
  resolution: "Enable logging: aws ec2 modify-verified-access-instance-logging-configuration --verified-access-instance-id <id> --access-logs CloudWatchLogs={Enabled=true,LogGroup=<group>}."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-instance-logging-configurations: GREEN — read-only config check"
  - "logs filter-log-events: GREEN — read-only log analysis"
  - "modify-verified-access-instance-logging-configuration: YELLOW — changes logging setup"
  - "logs delete-log-group: RED — permanently destroys log data"
```

## Escalation Conditions

- Logging is enabled but no logs are being written
- Log volume is excessive causing cost concerns
- Access logs show patterns indicating security incidents
- Log delivery failures to S3 or Kinesis Data Firehose

## Data Sensitivity

- Logging configuration: LOW — reveals log destination settings
- Access logs: HIGH — contain user identity, device posture, source IPs, and access decisions
- Log group metadata: LOW — reveals log group names and retention
- CloudWatch Insights queries: LOW — reveals analysis patterns

## Prohibited Actions

- NEVER delete access log groups without archiving data first
- NEVER share access logs containing user PII without authorization
- NEVER disable logging on production Verified Access instances
- NEVER set log retention to less than your compliance requirements

## Phase 3 — Rollback

1. If logging was disabled, re-enable it with previous configuration
2. If log group was deleted, create a new one and update logging configuration
3. If log retention was changed, revert to previous retention period
4. Verify logs are being written after configuration changes
5. Document logging configuration for disaster recovery

## Output Format

```yaml
root_cause: "access_logging — <specific_cause>"
evidence:
  - type: log_analysis
    content: "<logging configuration and log entries>"
severity: MEDIUM
mitigation:
  immediate: "Enable or fix logging"
  long_term: "Implement log analysis automation and monitoring"
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
