---
title: "G1 — App Runner Logging Issues"
description: "Diagnose CloudWatch Logs configuration and missing log problems"
status: active
severity: MEDIUM
triggers:
  - "logs missing"
  - "no logs"
  - "CloudWatch Logs"
  - "log group"
  - "log stream"
  - "cannot see logs"
owner: devops-agent
objective: "Identify and fix logging configuration issues for App Runner services"
context: "App Runner automatically sends application stdout/stderr to CloudWatch Logs. Log groups are created automatically in the format /aws/apprunner/<service-name>/<service-id>/application (for app logs) and /aws/apprunner/<service-name>/<service-id>/service (for service events). Logs are enabled by default and cannot be disabled. Common issues include looking in the wrong log group, log retention settings, and application not writing to stdout/stderr."
---

## Phase 1 — Triage

MUST:
- Check the service name and ID: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.{Name:ServiceName,Id:ServiceId}'`
- List log groups for the service: `aws logs describe-log-groups --log-group-name-prefix /aws/apprunner/<service-name>`
- List log streams in the application log group: `aws logs describe-log-streams --log-group-name /aws/apprunner/<service-name>/<service-id>/application --order-by LastEventTime --descending --limit 5`
- Get recent log events: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream> --limit 100`
- Check service event logs: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/service --log-stream-name <stream>`

SHOULD:
- Verify the application writes to stdout/stderr (not just to files)
- Check log retention settings: `aws logs describe-log-groups --log-group-name-prefix /aws/apprunner/<service-name> --query 'logGroups[*].{Name:logGroupName,Retention:retentionInDays}'`
- Check if the correct region is being queried (logs are in the same region as the service)

MAY:
- Set up CloudWatch Logs Insights queries for structured log analysis
- Check if log group has a resource policy blocking access
- Verify IAM permissions for the user/role querying logs

## Phase 2 — Remediate

MUST:
- Ensure the application writes logs to stdout and stderr (not to files inside the container)
- Use the correct log group name format: /aws/apprunner/<service-name>/<service-id>/application
- Query logs in the same AWS region as the App Runner service

SHOULD:
- Set appropriate log retention to manage costs: `aws logs put-retention-policy --log-group-name <log-group> --retention-in-days 30`
- Use structured logging (JSON) for easier querying with CloudWatch Logs Insights
- Separate application logs (stdout) from error logs (stderr)

MAY:
- Set up CloudWatch Logs subscription filters for real-time log processing
- Export logs to S3 for long-term retention
- Create CloudWatch Logs Insights saved queries for common troubleshooting patterns

## Common Issues

- symptoms: "No logs visible in CloudWatch"
  diagnosis: "Looking in the wrong log group or wrong region."
  resolution: "Use the correct format: /aws/apprunner/<service-name>/<service-id>/application in the service's region."

- symptoms: "Application logs are empty but service is running"
  diagnosis: "Application writes logs to files instead of stdout/stderr."
  resolution: "Configure the application to log to stdout/stderr. Most frameworks have a console logger option."

- symptoms: "Old logs are missing"
  diagnosis: "Log retention policy is set to a short period and logs have expired."
  resolution: "Increase log retention: aws logs put-retention-policy --log-group-name <group> --retention-in-days 90."

## Output Format

```yaml
root_cause: "logging — <specific_cause>"
evidence:
  - type: log_group
    content: "<log group name and existence>"
  - type: log_streams
    content: "<available log streams>"
  - type: retention
    content: "<log retention setting>"
severity: MEDIUM
mitigation:
  immediate: "Fix log group reference or application logging configuration"
  long_term: "Implement structured logging and appropriate retention policies"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure app writes to stdout/stderr | YELLOW | Application code change |
| Use correct log group name format | GREEN | Configuration fix |
| Query logs in correct region | GREEN | Operational guidance |
| Set log retention policy | GREEN | Cost management |
| Use structured logging (JSON) | YELLOW | Application code change |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Application logs**: Application output and errors — INTERNAL
- **Log group configuration**: Log group names and retention — INTERNAL
- **Log content**: May contain sensitive application data — CONFIDENTIAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER set log retention to 0 (never expire) without cost approval

## Phase 3 — Rollback

1. Revert application logging changes and redeploy
2. Restore previous log retention policy
3. Verify logs are flowing to the correct log group
4. Remove any subscription filters added during troubleshooting
5. Confirm log-based alarms are still functioning

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
