---
title: "G2 — EventBridge Automation"
description: "Diagnose GuardDuty EventBridge integration and automated response issues"
status: active
severity: MEDIUM
triggers:
  - "eventbridge automation"
  - "automated response"
  - "guardduty event rule"
  - "finding notification"
  - "lambda response"
owner: devops-agent
objective: "Identify and resolve EventBridge automation issues for GuardDuty finding response"
context: "GuardDuty publishes findings to EventBridge, enabling automated response workflows. Issues arise from incorrect event patterns, missing EventBridge rules, target configuration errors, or IAM permission problems."
---

## Phase 1 — Triage

MUST:
- List EventBridge rules for GuardDuty: `aws events list-rules --name-prefix guardduty`
- Describe the rule: `aws events describe-rule --name <rule-name>`
- Check the event pattern matches GuardDuty finding events
- List rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify the target (Lambda, SNS, SQS, etc.) is configured correctly

SHOULD:
- Check EventBridge rule state (ENABLED/DISABLED)
- Verify the IAM role for EventBridge has permissions to invoke the target
- Check CloudWatch metrics for rule invocations: `aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name Invocations --dimensions Name=RuleName,Value=<rule-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

MAY:
- Check failed invocations: `aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name FailedInvocations --dimensions Name=RuleName,Value=<rule-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Test the rule with a sample event

## Phase 2 — Remediate

MUST:
- Fix the event pattern to match GuardDuty finding events
- Ensure the EventBridge rule is ENABLED
- Verify target permissions and configuration

SHOULD:
- Test the automation with a low-severity finding
- Set up dead-letter queues for failed target invocations
- Monitor rule invocation metrics

MAY:
- Create separate rules for different severity levels
- Implement multi-step response workflows using Step Functions

## Common Issues

- symptoms: "EventBridge rule exists but never triggers for GuardDuty findings"
  diagnosis: "The event pattern does not match GuardDuty finding events. The source should be 'aws.guardduty' and detail-type 'GuardDuty Finding'."
  resolution: "Update the event pattern: {\"source\":[\"aws.guardduty\"],\"detail-type\":[\"GuardDuty Finding\"]}."

- symptoms: "Rule triggers but target Lambda function fails"
  diagnosis: "The Lambda function has errors processing the GuardDuty finding event, or lacks permissions."
  resolution: "Check Lambda CloudWatch logs. Verify the function handles the GuardDuty event format correctly."

- symptoms: "Rule triggers but SNS notification not received"
  diagnosis: "The SNS topic policy does not allow EventBridge to publish, or subscriptions are not confirmed."
  resolution: "Update SNS topic policy to allow events.amazonaws.com to publish. Confirm subscriptions."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix EventBridge event pattern | YELLOW | Rule modification; reversible |
| Enable EventBridge rule | YELLOW | State change; reversible |
| Fix target permissions | YELLOW | Permission change; reversible |

## Escalation Conditions

- Detector configuration changes
- EventBridge rule changes affecting automated response

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-rule` | LOW | EventBridge rule configuration |
| `list-targets-by-rule` | LOW | Target configuration |
| CloudWatch invocation metrics | LOW | Aggregate metrics |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Event pattern fix | Revert via `put-rule` with previous pattern |
| Rule enablement | Disable via `disable-rule` |
| Target permission fix | Revert IAM policy |

## Output Format

```yaml
root_cause: "eventbridge_automation — <specific_cause>"
evidence:
  - type: rule
    content: "<EventBridge rule configuration>"
  - type: targets
    content: "<rule targets and status>"
  - type: metrics
    content: "<invocation and failure metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix the event pattern or target configuration"
  long_term: "Monitor automation metrics and implement error handling"
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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
