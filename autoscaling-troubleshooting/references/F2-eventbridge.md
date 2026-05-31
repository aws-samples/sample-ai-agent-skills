---
title: "F2 — EventBridge Integration Issues"
description: "Diagnose EventBridge not receiving Auto Scaling events"
status: active
severity: LOW
triggers:
  - "EventBridge not receiving events"
  - "Auto Scaling events missing"
  - "notification not sent"
  - "event rule not matching"
  - "scaling event not captured"
owner: devops-agent
objective: "Identify why EventBridge is not receiving Auto Scaling events"
context: "Auto Scaling publishes events to EventBridge for instance launches, terminations, failures, and lifecycle actions. Issues include: event rule pattern mismatch, target permissions, or event bus configuration."
---

## Phase 1 — Triage

MUST:
- Check EventBridge rules for Auto Scaling: `aws events list-rules --name-prefix <prefix> --query 'Rules[*].{Name:Name,State:State,EventPattern:EventPattern}'`
- Check rule event pattern matches Auto Scaling events: `aws events describe-rule --name <rule-name>`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify ASG notification configuration: `aws autoscaling describe-notification-configurations --auto-scaling-group-names <asg-name>`

SHOULD:
- Test event pattern with: `aws events test-event-pattern --event-pattern '<pattern>' --event '<test-event>'`
- Check target permissions (Lambda, SNS, SQS)
- Verify event bus (default vs custom)

MAY:
- Check CloudTrail for EventBridge delivery failures
- Review dead-letter queue for failed deliveries

## Phase 2 — Remediate

MUST:
- Fix event pattern to match Auto Scaling events: source should be "aws.autoscaling"
- Ensure rule is ENABLED
- Fix target permissions

SHOULD:
- Create correct rule: `aws events put-rule --name asg-events --event-pattern '{"source":["aws.autoscaling"],"detail-type":["EC2 Instance Launch Successful","EC2 Instance Terminate Successful","EC2 Instance Launch Unsuccessful"]}'`
- Add target with proper permissions

MAY:
- Set up dead-letter queue for failed deliveries
- Use input transformation to customize event payload

## Output Format

```yaml
root_cause: "eventbridge_issue — <specific_cause>"
evidence:
  - type: rule_config
    content: "<event rule and pattern>"
  - type: target_config
    content: "<rule targets>"
severity: LOW
mitigation:
  immediate: "Fix event pattern or target permissions"
  long_term: "Implement comprehensive event monitoring with DLQ"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix event pattern to match Auto Scaling events | GREEN | Event rule configuration — non-destructive |
| Ensure rule is ENABLED | GREEN | Enables monitoring — non-destructive |
| Fix target permissions | YELLOW | IAM/resource policy change |
| Set up dead-letter queue | GREEN | Error handling — non-destructive |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- EventBridge rule changes affect operational monitoring and alerting
- Target permission changes may affect other event-driven workflows

## Data Sensitivity

- EventBridge events contain Auto Scaling operational data
- Event patterns reveal what scaling events are monitored
- Target configurations reveal notification and automation endpoints

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER disable EventBridge rules that feed critical alerting systems
- NEVER modify event patterns without understanding downstream consumers

## Phase 3 — Rollback

- If event rule was modified: restore original event pattern
- If rule was enabled incorrectly: disable the rule
- If target permissions were changed: revert permission changes
- Document all EventBridge changes for operational audit trail

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
