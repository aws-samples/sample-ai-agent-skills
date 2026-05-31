---
title: "F2 — Notification Issues"
description: "Diagnose alarm notification delivery failures"
status: active
severity: MEDIUM
triggers:
  - "notification not sent"
  - "SNS not triggered"
  - "alarm action failed"
owner: devops-agent
objective: "Identify and resolve alarm notification delivery failures"
context: "Notification issues occur when: SNS topic permissions wrong, Lambda action fails, notification action not configured, or IAM role insufficient."
---

## Phase 1 — Triage

MUST:
- Check alarm notification action configuration
- Verify SNS topic exists and has correct permissions
- Check IAM role for alarm actions has sns:Publish permission
- Verify alarm is actually transitioning states

SHOULD:
- Check SNS delivery logs for failures
- Verify Lambda function if using Lambda action
- Check CloudTrail for alarm action API calls

MAY:
- Test SNS topic manually: `aws sns publish --topic-arn <arn> --message "test"`
- Check alarm state transition history

## Phase 2 — Remediate

MUST:
- Fix SNS topic permissions for SiteWise
- Grant IAM role sns:Publish permission
- Configure notification action on alarm

SHOULD:
- Test notification delivery end-to-end
- Set up monitoring for notification failures

## Output Format

```yaml
root_cause: "notification_issue — <specific_cause>"
evidence:
  - type: notification_config
    content: "<action configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions or action config"
  long_term: "Monitor notification delivery"
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
- Never delete asset models that have active assets associated with them
- Never modify measurement properties on assets receiving live industrial data
- Never remove gateway configurations without confirming data source migration
- Never alter transform or metric computations on production assets without testing
- Never delete portal or dashboard configurations used by operations teams

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous asset model configuration if property computations break
- Revert gateway data source settings if industrial data ingestion stops
- Re-enable previous alarm configurations if monitoring coverage is lost

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
