---
title: "F1 — Audit Logging"
description: "Diagnose audit logging gaps for AWS Chatbot actions including CloudTrail configuration and log analysis"
status: active
severity: MEDIUM
triggers: ["audit log missing", "no CloudTrail events for chatbot", "cannot track chat commands", "compliance logging"]
owner: devops-agent
objective: "Resolve audit logging gaps for AWS Chatbot actions and commands"
context: "CloudTrail logs Chatbot API calls and command executions but NOT chat message content. Chatbot-specific logging can be enabled per channel configuration. Audit completeness depends on CloudTrail and Chatbot logging configuration."
---

## Phase 1 — Triage

MUST:
- Check CloudTrail for Chatbot events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 20`
- Verify CloudTrail is enabled and logging: `aws cloudtrail get-trail-status --name <trail-name>`
- Check if Chatbot logging is enabled in channel configuration

SHOULD:
- Review CloudTrail event history for specific time ranges
- Check if the trail covers the region where Chatbot is configured
- Verify CloudTrail log delivery to S3 is working

MAY:
- Check CloudWatch Logs if Chatbot logging is configured to CloudWatch
- Review S3 bucket for CloudTrail log files
- Set up CloudTrail Insights for anomaly detection

## Phase 2 — Remediate

MUST:
- Enable CloudTrail if not active
- Ensure CloudTrail covers the Chatbot region
- Enable Chatbot-specific logging in channel configurations

SHOULD:
- Configure CloudTrail to log management events (includes Chatbot API calls)
- Set up log analysis for Chatbot command audit
- Implement log retention policies

MAY:
- Create CloudWatch alarms for specific Chatbot actions
- Set up Athena queries for CloudTrail log analysis
- Implement centralized logging across accounts

## Safety Ratings

```
safety_ratings:
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
  - "cloudtrail get-trail-status: GREEN — read-only trail status"
  - "cloudtrail create-trail: YELLOW — creates new audit trail"
  - "update-slack-channel-configuration (logging): YELLOW — modifies logging settings"
```

## Escalation Conditions

- Compliance audit requires Chatbot action logs that are not available
- CloudTrail is disabled or misconfigured in the Chatbot region
- Audit gap covers a period with suspected unauthorized actions
- Log retention does not meet compliance requirements

## Data Sensitivity

- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and actions
- `aws cloudtrail get-trail-status`: LOW — returns trail configuration status
- CloudTrail log files: HIGH — contain detailed API call records
- Chatbot logs: MEDIUM — contain command execution details

## Prohibited Actions

- NEVER disable CloudTrail to troubleshoot logging issues
- NEVER delete CloudTrail log files
- NEVER modify log retention policies without compliance team approval
- NEVER share audit logs containing IAM principal details externally

## Phase 3 — Rollback

1. If CloudTrail configuration was changed, restore previous settings
2. If Chatbot logging was modified, revert to previous configuration
3. Verify audit logging resumes correctly
4. Check for any gap in log coverage during the change
5. Document the logging configuration change

## Common Issues

- symptoms: "No CloudTrail events for Chatbot actions"
  diagnosis: "CloudTrail is not enabled, not covering the Chatbot region, or not logging management events."
  resolution: "Enable CloudTrail with management event logging in the Chatbot region."

- symptoms: "CloudTrail shows API calls but not chat command details"
  diagnosis: "CloudTrail logs API calls but not the chat-level command context."
  resolution: "Enable Chatbot-specific logging in the channel configuration for detailed command audit."

- symptoms: "Audit logs missing for a specific time period"
  diagnosis: "CloudTrail had a delivery failure or was temporarily disabled."
  resolution: "Check CloudTrail delivery status. Review S3 bucket permissions. Verify trail was active during the period."

## Output Format

```yaml
root_cause: "audit_logging — <specific_cause>"
evidence:
  - type: logging_error
    content: "<CloudTrail and logging configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix audit logging configuration"
  long_term: "Implement comprehensive logging and monitoring"
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
