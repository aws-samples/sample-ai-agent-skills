---
title: "E1 — Log Configuration"
description: "Diagnose MediaTailor logging configuration and CloudWatch Logs issues"
status: active
severity: MEDIUM
triggers: ["no logs", "logging not working", "CloudWatch Logs empty", "cannot find MediaTailor logs"]
owner: devops-agent
objective: "Resolve log configuration issues to enable debugging and monitoring"
context: "MediaTailor logging is not enabled by default. Log percentages (1-100%) must be configured per playback configuration. Logs go to CloudWatch Logs with the log group /aws/mediatailor/<config-name>."
---

## Phase 1 — Triage

MUST:
- Check if logging is configured: `aws mediatailor get-playback-configuration --Name <config-name> --query 'LogConfiguration'`
- Verify CloudWatch Logs log group exists: `aws logs describe-log-groups --log-group-name-prefix /aws/mediatailor/<config-name>`
- Check IAM permissions for MediaTailor to write to CloudWatch Logs

SHOULD:
- Verify log percentage is greater than 0
- Check for recent log events: `aws logs filter-log-events --log-group-name /aws/mediatailor/<config-name> --start-time <epoch-ms> --limit 10`

MAY:
- Review CloudWatch Logs retention settings
- Check for log group resource policy allowing MediaTailor

## Phase 2 — Remediate

MUST:
- Enable logging by updating configuration with log percentage: `aws mediatailor put-playback-configuration --Name <config-name> --LogConfiguration PercentEnabled=5`
- Ensure CloudWatch Logs resource policy allows mediatailor.amazonaws.com
- Verify IAM service-linked role has logs:CreateLogGroup and logs:PutLogEvents permissions

SHOULD:
- Set appropriate log percentage (1-5% for production, higher for debugging)
- Configure log retention period to manage costs

MAY:
- Set up CloudWatch Logs Insights queries for common error patterns
- Create CloudWatch alarms based on log filter metrics

## Safety Ratings

```
safety_ratings:
  - "get-playback-configuration: GREEN — read-only configuration retrieval"
  - "describe-log-groups: GREEN — read-only log group enumeration"
  - "filter-log-events: GREEN — read-only log query"
  - "put-playback-configuration log config: YELLOW — changes logging behavior"
  - "put-resource-policy on logs: YELLOW — modifies CloudWatch Logs access"
```

## Escalation Conditions

- Logging cannot be enabled due to CloudWatch Logs service limits
- Log group resource policy requires changes managed by security team
- High log volume causing unexpected CloudWatch Logs costs
- Logs needed urgently for live incident but logging was never configured

## Data Sensitivity

- Log configuration: LOW — log percentage and group name
- CloudWatch Logs content: HIGH — may contain session IDs, ADS URLs with parameters, and error details
- Log group policies: MEDIUM — reveal service principals with write access
- Log retention settings: LOW — configuration metadata

## Prohibited Actions

- NEVER set log percentage to 100% on high-traffic production configurations without cost analysis
- NEVER delete CloudWatch Logs log groups containing incident investigation data
- NEVER share log entries containing ADS URLs with authentication tokens
- NEVER modify CloudWatch Logs resource policies to allow unauthorized principals

## Phase 3 — Rollback

1. If log percentage was increased too high, reduce it: update configuration with lower percentage
2. If log group was accidentally deleted, re-enable logging (MediaTailor recreates the group)
3. If resource policy was changed incorrectly, restore previous policy: `aws logs put-resource-policy`
4. Verify logs are flowing after rollback by checking for new log events

## Common Issues

- symptoms: "No CloudWatch Logs log group exists for the configuration"
  diagnosis: "Logging was never enabled on the playback configuration."
  resolution: "Enable logging with put-playback-configuration including LogConfiguration with PercentEnabled > 0."

- symptoms: "Log group exists but no log events"
  diagnosis: "Log percentage is 0 or very low with insufficient traffic to generate sampled logs."
  resolution: "Increase log percentage temporarily. Verify configuration has active playback sessions."

## Output Format

```yaml
root_cause: "log_configuration — <specific_cause>"
evidence:
  - type: log_config
    content: "<log configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Enable or fix logging configuration"
  long_term: "Include log configuration in IaC templates"
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
