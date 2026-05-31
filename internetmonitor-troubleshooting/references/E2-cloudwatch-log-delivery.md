---
title: "E2 — CloudWatch Log Delivery"
description: "Diagnose CloudWatch Logs delivery failures for Internet Monitor"
status: active
severity: MEDIUM
triggers: ["CloudWatch logs missing", "log group empty", "no logs in CloudWatch", "log delivery error"]
owner: devops-agent
objective: "Resolve CloudWatch Logs delivery issues for Internet Monitor data export"
context: "Internet Monitor can deliver logs to CloudWatch Logs for real-time analysis and alerting. Delivery must be explicitly configured with correct log group permissions."
---

## Phase 1 — Triage

MUST:
- Check monitor log delivery configuration: `aws internetmonitor get-monitor --monitor-name <name> --query 'InternetMeasurementsLogDelivery'`
- Verify log group exists: `aws logs describe-log-groups --log-group-name-prefix <log-group-name>`
- Check log group for recent log streams: `aws logs describe-log-streams --log-group-name <name> --order-by LastEventTime --descending --limit 5`

SHOULD:
- Verify IAM permissions allow Internet Monitor to write to the log group
- Check the log group's resource policy: `aws logs describe-resource-policies`
- Verify the log group is in the same region as the monitor

MAY:
- Check CloudTrail for PutLogEvents failures from Internet Monitor
- Review log group retention settings

## Phase 2 — Remediate

MUST:
- Create log group if missing: `aws logs create-log-group --log-group-name /aws/internetmonitor/<monitor-name>`
- Configure log delivery on the monitor if not set
- Set resource policy on the log group to allow Internet Monitor service principal

SHOULD:
- Set appropriate log retention period: `aws logs put-retention-policy --log-group-name <name> --retention-in-days 30`
- Configure metric filters for automated alerting on log data

MAY:
- Set up CloudWatch Logs Insights queries for traffic analysis
- Configure log group subscription filters for downstream processing

## Common Issues

- symptoms: "Log group exists but no log streams from Internet Monitor"
  diagnosis: "Log group resource policy may not allow Internet Monitor to create log streams and put events."
  resolution: "Set resource policy: `aws logs put-resource-policy --policy-name InternetMonitorPolicy --policy-document '{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"internetmonitor.amazonaws.com\"},\"Action\":[\"logs:CreateLogStream\",\"logs:PutLogEvents\"],\"Resource\":\"<log-group-arn>\"}]}'`"

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "describe-log-groups: GREEN — read-only log group listing"
  - "describe-log-streams: GREEN — read-only log stream listing"
  - "put-resource-policy: YELLOW — modifies log group access permissions"
```

## Escalation Conditions

- Log delivery configured correctly but no logs appearing after 1 hour
- Log group receiving excessive volume causing cost concerns
- Resource policy changes affect other services writing to the same log group
- Log data format does not match expected schema for downstream processing

## Data Sensitivity

- `aws logs describe-log-groups`: LOW — returns log group metadata
- `aws logs describe-resource-policies`: MEDIUM — reveals access permissions
- Internet Monitor logs in CloudWatch: MEDIUM — contain traffic patterns and city-network data

## Prohibited Actions

- NEVER delete a log group that may contain data needed for other purposes
- NEVER set overly broad resource policies (logs:* for all principals)
- NEVER reduce log retention below compliance requirements
- NEVER modify resource policies without reviewing impact on other services

## Phase 3 — Rollback

1. If log delivery was enabled in error, disable it on the monitor
2. If resource policy was modified, revert to previous policy: `aws logs delete-resource-policy --policy-name <name>` and recreate original
3. If log retention was changed, restore original retention period
4. If a log group was created in error, delete only if confirmed empty and not needed
5. Document all CloudWatch Logs and monitor configuration changes

## Output Format

```yaml
root_cause: "cloudwatch_log_delivery — <specific_cause>"
evidence:
  - type: delivery_error
    content: "<CloudWatch Logs and delivery configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix CloudWatch Logs delivery configuration"
  long_term: "Automate log group and delivery setup via IaC"
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
