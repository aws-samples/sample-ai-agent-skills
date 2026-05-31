---
title: "F2 — EventBridge Integration"
description: "Diagnose Macie EventBridge notification issues"
status: active
severity: MEDIUM
triggers:
  - "no Macie notifications"
  - "EventBridge not triggering"
  - "Macie events missing"
owner: devops-agent
objective: "Ensure Macie findings trigger EventBridge rules correctly"
context: "Macie publishes finding events to EventBridge for automation and notifications. Events use source 'aws.macie' and detail-type 'Macie Finding'. Rules must match this pattern to trigger targets."
---

## Phase 1 — Triage

MUST:
- List EventBridge rules: `aws events list-rules --query 'Rules[?contains(Name, `macie`) || contains(Description, `Macie`)]'`
- Check rule event pattern: `aws events describe-rule --name <rule-name>`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify rule is enabled

SHOULD:
- Verify event pattern uses source 'aws.macie'
- Check target permissions
- Test with sample event

MAY:
- Check CloudWatch metrics for rule invocations
- Review dead-letter queue

## Phase 2 — Remediate

MUST:
- Create/fix rule with pattern: source 'aws.macie', detail-type 'Macie Finding'
- Ensure rule is ENABLED
- Configure target with proper permissions

SHOULD:
- Filter by severity for targeted notifications
- Set up SNS for email/Slack alerts
- Test end-to-end notification flow

MAY:
- Create Lambda targets for automated remediation
- Set up separate rules for different finding types

## Common Issues

- symptoms: "Rule never triggers"
  diagnosis: "Event pattern doesn't match Macie event format."
  resolution: "Use source 'aws.macie' and detail-type 'Macie Finding'."

- symptoms: "Rule triggers but no notification"
  diagnosis: "Target permissions missing."
  resolution: "Add resource-based policy allowing EventBridge to invoke target."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create/fix EventBridge rule | GREEN | Event routing configuration |
| Configure target permissions | YELLOW | Resource policy changes |
| Test notification flow | GREEN | Verification only |

## Escalation Conditions

- Security notification pipeline broken — escalate to security operations
- Automated remediation targets failing — escalate to automation team
- Compliance alerting not working — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws events describe-rule` | MEDIUM | Exposes event pattern configuration |
| `aws events list-targets-by-rule` | MEDIUM | Exposes target configuration |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** disable EventBridge rules for security notifications without approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created EventBridge rule | Delete: `aws events delete-rule --name <name>` |
| Fixed event pattern | Restore previous event pattern |
| Configured target | Remove target: `aws events remove-targets` |

## Output Format

```yaml
root_cause: "eventbridge — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule pattern and state>"
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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
