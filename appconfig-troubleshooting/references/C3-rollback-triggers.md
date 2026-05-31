---
title: "C3 — Rollback Triggers"
description: "Diagnose rollback trigger configuration and behavior"
status: active
severity: HIGH
triggers:
  - "rollback not triggering"
  - "unexpected rollback"
  - "monitor alarm"
owner: devops-agent
objective: "Ensure rollback triggers work correctly"
context: "Rollback triggers use CloudWatch alarms configured as environment monitors. When an alarm enters ALARM state during deployment, AppConfig rolls back."
---

## Phase 1 — Triage

MUST:
- Get environment monitors: `aws appconfig get-environment --application-id <app-id> --environment-id <env-id>`
- Check CloudWatch alarm state: `aws cloudwatch describe-alarms --alarm-names <alarm-name>`
- Verify the alarm ARN in the environment matches an existing alarm
- Check the IAM role for monitoring has cloudwatch:DescribeAlarms permission

SHOULD:
- Check alarm history during the deployment window
- Verify the alarm metric and threshold are appropriate

MAY:
- Review deployment history for rollback patterns

## Phase 2 — Remediate

MUST:
- Fix alarm ARN if it references a non-existent alarm
- Ensure the monitoring IAM role has correct permissions
- If alarm is too sensitive: adjust threshold to avoid false rollbacks

SHOULD:
- Use composite alarms for more nuanced rollback triggers
- Set alarm evaluation periods appropriate for the deployment duration

MAY:
- Add multiple monitors for different failure signals

safety_ratings:
  - "Checking alarm state: GREEN"
  - "Adjusting alarm threshold: YELLOW"
  - "Removing a monitor: RED — removes rollback protection"

escalation_conditions:
  - "Production environment has no working rollback triggers"
  - "Rollback triggers firing on every deployment (false positives)"

data_sensitivity:
  - command: "describe-alarms"
    sensitivity: LOW
    contains: "Alarm names, thresholds, states"

prohibited_actions:
  - "NEVER remove all monitors from a production environment"
  - "NEVER set alarm thresholds so high they never trigger"

## Phase 3 — Rollback

- If monitor configuration was changed, restore previous monitor settings
- If alarm threshold was adjusted, document the original threshold

## Common Issues

- symptoms: "Rollback doesn't trigger despite application errors"
  diagnosis: "CloudWatch alarm not configured as environment monitor, or alarm metric doesn't capture the error"
  resolution: "Add the alarm as an environment monitor and verify the metric reflects application health"

- symptoms: "Every deployment rolls back immediately"
  diagnosis: "CloudWatch alarm was already in ALARM state before deployment started"
  resolution: "Resolve the alarm condition before starting the deployment"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: monitor_config
    content: "<environment monitors and alarm state>"
severity: HIGH
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
