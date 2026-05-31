---
title: "F2 — EventBridge Integration"
description: "Diagnose CloudTrail EventBridge integration issues"
status: active
severity: MEDIUM
triggers:
  - "EventBridge not triggering"
  - "CloudTrail events not in EventBridge"
  - "rule not matching"
owner: devops-agent
objective: "Ensure CloudTrail events trigger EventBridge rules correctly"
context: "CloudTrail management events are automatically sent to EventBridge (near-real-time). EventBridge rules can match specific API calls for automated responses. Data events are NOT sent to EventBridge by default."
---

## Phase 1 — Triage

MUST:
- List relevant EventBridge rules: `aws events list-rules --query 'Rules[*].{Name:Name,State:State}'`
- Check rule event pattern: `aws events describe-rule --name <rule-name>`
- Check rule targets: `aws events list-targets-by-rule --rule <rule-name>`
- Verify rule is ENABLED

SHOULD:
- Verify event pattern matches CloudTrail event format (source: aws.<service>, detail-type: AWS API Call via CloudTrail)
- Check target permissions
- Test with a matching API call

MAY:
- Check CloudWatch metrics for rule invocations
- Review dead-letter queue for failed deliveries

## Phase 2 — Remediate

MUST:
- Fix event pattern to match CloudTrail format
- Ensure rule is ENABLED
- Configure target with proper permissions

SHOULD:
- Use specific event patterns (eventName, eventSource) for targeted matching
- Test end-to-end with a real API call

MAY:
- Create rules for security-critical API calls
- Set up automated remediation with Lambda targets

## Common Issues

- symptoms: "Rule never triggers on CloudTrail events"
  diagnosis: "Event pattern doesn't match CloudTrail event format."
  resolution: "Use source 'aws.<service>' and detail-type 'AWS API Call via CloudTrail'."

- symptoms: "Data events not triggering rules"
  diagnosis: "Data events are not sent to EventBridge by default."
  resolution: "Data events require CloudTrail trail with data events enabled. Use S3 event notifications instead."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix event pattern | GREEN | Rule configuration change, no state impact |
| Enable rule | GREEN | Activates event matching |
| Configure target with permissions | YELLOW | Resource policy changes |

## Escalation Conditions

- Security-critical EventBridge rules not triggering — escalate to security operations
- Automated remediation Lambda targets failing — escalate to automation team
- EventBridge rules for compliance monitoring broken — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws events describe-rule` | HIGH | Exposes event pattern matching CloudTrail events |
| `aws events list-targets-by-rule` | MEDIUM | Exposes target configuration |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** disable EventBridge rules for security-critical events without approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed event pattern | Restore previous event pattern |
| Enabled rule | Disable rule: `aws events disable-rule --name <name>` |
| Configured target | Remove target: `aws events remove-targets --rule <name> --ids <target-id>` |

## Output Format

```yaml
root_cause: "eventbridge — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule event pattern>"
severity: MEDIUM
mitigation:
  immediate: "Fix event pattern and target"
  long_term: "Set up comprehensive event-driven automation"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
