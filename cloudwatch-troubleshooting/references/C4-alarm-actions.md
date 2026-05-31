---
title: "C4 — Alarm Actions"
description: "Diagnose alarm action failures (SNS, Auto Scaling, EC2, Lambda)"
status: active
severity: HIGH
triggers:
  - "alarm action"
  - "SNS notification"
  - "alarm not sending"
  - "auto scaling action"
  - "EC2 action"
  - "alarm notification"
owner: devops-agent
objective: "Identify why alarm actions are not executing when the alarm state changes"
context: "Alarm actions trigger on state transitions (OK→ALARM, ALARM→OK, any→INSUFFICIENT_DATA). Actions can target SNS topics, Auto Scaling policies, EC2 actions (stop, terminate, reboot, recover), or Lambda functions. Failures are caused by disabled actions, missing SNS permissions, deleted targets, or the alarm not actually transitioning state."
---

## Phase 1 — Triage

MUST:
- Check alarm actions are configured: `aws cloudwatch describe-alarms --alarm-names <name> --query 'MetricAlarms[].{AlarmActions:AlarmActions,OKActions:OKActions,ActionsEnabled:ActionsEnabled}'`
- Verify ActionsEnabled is true (actions can be globally disabled on an alarm)
- Verify the action target exists (SNS topic, Auto Scaling policy, etc.)
- Check alarm state history for recent transitions: `aws cloudwatch describe-alarm-history --alarm-name <name> --history-item-type Action`

SHOULD:
- Check SNS topic subscription status: `aws sns list-subscriptions-by-topic --topic-arn <arn>`
- Verify SNS topic policy allows CloudWatch to publish: `aws sns get-topic-attributes --topic-arn <arn>`
- Check if the alarm transitioned state (actions only fire on transitions, not while staying in ALARM)

MAY:
- Check SNS delivery logs for failures
- Verify email subscription is confirmed (pending subscriptions don't receive messages)

## Phase 2 — Remediate

MUST:
- Enable actions if disabled: `aws cloudwatch enable-alarm-actions --alarm-names <name>`
- Fix SNS topic policy to allow `sns:Publish` from `cloudwatch.amazonaws.com`
- Ensure action target ARNs are valid and resources exist

SHOULD:
- Confirm SNS email subscriptions (check spam folder for confirmation email)
- Add both AlarmActions and OKActions for complete notification coverage
- Test by manually setting alarm state: `aws cloudwatch set-alarm-state --alarm-name <name> --state-value ALARM --state-reason "Testing"`

MAY:
- Set up SNS delivery status logging for debugging
- Use SNS message filtering for targeted notifications

## Common Issues

- symptoms: "Alarm enters ALARM but no SNS notification received"
  diagnosis: "ActionsEnabled is false, or SNS topic policy doesn't allow CloudWatch."
  resolution: "Enable alarm actions. Add CloudWatch service principal to SNS topic policy."

- symptoms: "First alarm notification received but subsequent ones not"
  diagnosis: "Alarm stays in ALARM state. Actions only fire on state transitions."
  resolution: "This is expected behavior. Actions fire on OK→ALARM transition, not while in ALARM."

- symptoms: "EC2 recover action not working"
  diagnosis: "EC2 recover action only works on instances with instance store or EBS-backed in supported instance types."
  resolution: "Verify instance type supports recovery. Check EC2 status check alarm configuration."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable alarm actions | YELLOW | May trigger pending notifications immediately |
| Fix SNS topic policy | YELLOW | Topic policy changes affect all publishers |
| Test alarm with set-alarm-state | YELLOW | Triggers alarm actions; use only in non-production or with approval |

## Escalation Conditions

- Production alarm actions not firing during incidents — escalate immediately to operations team
- SNS topic policy changes — escalate to topic owner
- EC2 recover action not working — escalate to infrastructure team
- Auto Scaling actions not triggering — escalate to scaling policy owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudwatch describe-alarms` (actions) | MEDIUM | Exposes alarm actions and SNS topic ARNs |
| `aws sns list-subscriptions-by-topic` | MEDIUM | Exposes notification endpoints (emails, phone numbers) |
| `aws sns get-topic-attributes` | MEDIUM | Exposes topic policy |
| `aws cloudwatch describe-alarm-history` (Action) | MEDIUM | Exposes action execution history |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** use set-alarm-state on production alarms without approval (triggers real actions)

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled alarm actions | Disable: `aws cloudwatch disable-alarm-actions --alarm-names <name>` |
| Fixed SNS topic policy | Restore previous topic policy |
| Set alarm state for testing | Set alarm back to OK: `aws cloudwatch set-alarm-state --state-value OK` |

## Output Format

```yaml
root_cause: "alarm_actions — <specific_cause>"
evidence:
  - type: alarm_config
    content: "<alarm actions and enabled status>"
  - type: action_target
    content: "<SNS topic or action target status>"
severity: HIGH
mitigation:
  immediate: "Enable actions, fix SNS permissions, or verify target exists"
  long_term: "Test alarm actions during change windows, monitor SNS delivery"
```

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
