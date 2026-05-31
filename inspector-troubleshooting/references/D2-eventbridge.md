---
title: "D2 — EventBridge Integration"
description: "Diagnose Inspector EventBridge notification issues"
status: active
severity: MEDIUM
triggers:
  - "no notifications"
  - "EventBridge not triggering"
  - "Inspector events missing"
  - "alert not received"
owner: devops-agent
objective: "Ensure Inspector findings trigger EventBridge rules correctly"
context: "Inspector sends finding events to EventBridge for automation and notifications. Events include finding creation, state changes, and initial/aggregate scans. EventBridge rules must match the Inspector event pattern to trigger targets."
---

## Phase 1 — Triage

MUST:
- List EventBridge rules for Inspector: `aws events list-rules --query 'Rules[?contains(Description, `Inspector`) || contains(Name, `inspector`)]'`
- Check rule event pattern: `aws events describe-rule --name <rule-name> --query 'EventPattern'`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Check rule state: `aws events describe-rule --name <rule-name> --query 'State'`

SHOULD:
- Verify event pattern matches Inspector event format (source: aws.inspector2)
- Check target permissions (SNS topic policy, Lambda invoke permissions)
- Test with a sample event: `aws events put-events --entries '[{"Source":"aws.inspector2","DetailType":"Inspector2 Finding","Detail":"{}"}]'`

MAY:
- Check CloudWatch metrics for rule invocations
- Review dead-letter queue for failed deliveries

## Phase 2 — Remediate

MUST:
- Create/fix EventBridge rule with correct pattern: source `aws.inspector2`, detail-type `Inspector2 Finding`
- Ensure rule is ENABLED
- Configure target with proper permissions

SHOULD:
- Filter events by severity for targeted notifications
- Set up SNS topic for email/Slack notifications
- Test the complete notification pipeline

MAY:
- Create separate rules for different severity levels
- Configure Lambda targets for automated remediation

## Common Issues

- symptoms: "EventBridge rule exists but never triggers"
  diagnosis: "Event pattern doesn't match Inspector event format."
  resolution: "Use source 'aws.inspector2' and detail-type 'Inspector2 Finding'."

- symptoms: "Rule triggers but target doesn't execute"
  diagnosis: "Target lacks permissions (SNS policy, Lambda permission)."
  resolution: "Add resource-based policy allowing EventBridge to invoke the target."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create/fix EventBridge rule | GREEN | Event routing configuration |
| Configure target permissions | YELLOW | Resource policy changes |
| Test notification pipeline | GREEN | Verification only |

## Escalation Conditions

- Security notification pipeline broken — escalate to security operations
- Automated remediation Lambda targets failing — escalate to automation team
- EventBridge rules for compliance alerting not working — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws events describe-rule` | MEDIUM | Exposes event pattern configuration |
| `aws events list-targets-by-rule` | MEDIUM | Exposes target configuration |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** disable EventBridge rules for security notifications without approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created EventBridge rule | Delete the rule: `aws events delete-rule --name <name>` |
| Fixed event pattern | Restore previous event pattern |
| Configured target | Remove target: `aws events remove-targets` |

## Output Format

```yaml
root_cause: "eventbridge — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule event pattern and state>"
  - type: target_config
    content: "<target configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix event pattern and target permissions"
  long_term: "Set up comprehensive notification pipeline"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
