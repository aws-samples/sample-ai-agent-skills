---
title: "F1 — Custom Actions"
description: "Diagnose Security Hub custom action configuration and EventBridge integration issues"
status: active
severity: MEDIUM
triggers:
  - "custom action"
  - "action target"
  - "custom action not working"
  - "action eventbridge"
  - "manual action"
owner: devops-agent
objective: "Identify and resolve custom action configuration and EventBridge integration issues"
context: "Custom actions in Security Hub send events to EventBridge when triggered. They require corresponding EventBridge rules with targets to perform actual work. Without EventBridge rules, custom actions do nothing. Custom actions can be triggered manually from the console or via automation."
---

## Phase 1 — Triage

MUST:
- List custom actions: `aws securityhub describe-action-targets`
- Check EventBridge rules for custom actions: `aws events list-rules --name-prefix securityhub`
- Verify EventBridge rule event pattern matches the custom action ARN
- Check EventBridge rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify the EventBridge rule is ENABLED

SHOULD:
- Check EventBridge invocation metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name Invocations --dimensions Name=RuleName,Value=<rule-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Verify the target (Lambda, SNS, etc.) is configured correctly
- Check for failed invocations

MAY:
- Test the custom action with a sample finding
- Check CloudTrail for custom action events

## Phase 2 — Remediate

MUST:
- Create EventBridge rules for custom actions that lack them
- Fix event pattern to match: `{"source":["aws.securityhub"],"detail-type":["Security Hub Findings - Custom Action"],"resources":["<custom-action-arn>"]}`
- Verify the target has proper permissions

SHOULD:
- Test the end-to-end flow with a test finding
- Set up dead-letter queues for failed target invocations

MAY:
- Create multiple custom actions for different response workflows
- Document custom action workflows for the team

## Common Issues

- symptoms: "Custom action created but nothing happens when triggered"
  diagnosis: "No EventBridge rule exists for the custom action, or the rule is disabled."
  resolution: "Create an EventBridge rule matching the custom action ARN with an appropriate target."

- symptoms: "EventBridge rule exists but target not invoked"
  diagnosis: "The event pattern does not match the custom action event, or the target permissions are incorrect."
  resolution: "Verify the event pattern includes the correct custom action ARN. Check target IAM permissions."

- symptoms: "Custom action triggers but Lambda target fails"
  diagnosis: "The Lambda function has errors processing the Security Hub finding event."
  resolution: "Check Lambda CloudWatch logs. Verify the function handles the Security Hub event format."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create EventBridge rules | YELLOW | New resource; can be deleted |
| Fix event pattern | YELLOW | Rule modification; reversible |
| Fix target permissions | YELLOW | Permission change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Custom action changes affecting security workflows

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-action-targets` | LOW | Custom action configuration |
| `list-rules` (EventBridge) | LOW | Rule configuration |
| CloudWatch invocation metrics | LOW | Aggregate metrics |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| EventBridge rule creation | Delete rule via `delete-rule` |
| Event pattern fix | Revert via `put-rule` |
| Target permission fix | Revert IAM policy |

## Output Format

```yaml
root_cause: "custom_action — <specific_cause>"
evidence:
  - type: action_targets
    content: "<custom action configuration>"
  - type: eventbridge_rules
    content: "<EventBridge rule and target configuration>"
  - type: metrics
    content: "<invocation metrics>"
severity: MEDIUM
mitigation:
  immediate: "Create or fix EventBridge rules for custom actions"
  long_term: "Monitor custom action workflows and implement error handling"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
