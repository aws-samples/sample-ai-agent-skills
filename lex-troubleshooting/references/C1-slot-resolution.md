---
title: "C1 — Slot Resolution Issues"
description: "Diagnose slot filling and value resolution problems"
status: active
severity: MEDIUM
triggers:
  - "slot not filled"
  - "wrong slot value"
  - "slot elicitation failing"
  - "slot resolution error"
owner: devops-agent
objective: "Identify and resolve slot filling issues"
context: "Slots capture specific values from user input. Issues arise from incorrect slot types, missing elicitation prompts, or slot priority misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check slot configuration: `aws lexv2-models describe-slot --bot-id <bot-id> --bot-version DRAFT --locale-id en_US --intent-id <intent-id> --slot-id <slot-id>`
- Test slot filling: `aws lexv2-runtime recognize-text --bot-id <bot-id> --bot-alias-id <alias-id> --locale-id en_US --session-id test-$(date +%s) --text "<utterance with slot value>"`
- Verify slot type matches expected input
- Check slot priority order in intent

SHOULD:
- Verify elicitation prompt is configured for required slots
- Check if slot is marked as required or optional
- Test with various input formats for the slot type

MAY:
- Check slot type values for custom slot types
- Review slot validation settings

## Phase 2 — Remediate

MUST:
- Use correct slot type for expected input (AMAZON.Date for dates, etc.)
- Configure elicitation prompts for required slots
- Set appropriate slot priorities

SHOULD:
- Add slot values in sample utterances with {SlotName} syntax
- Configure slot validation with Lambda code hook

MAY:
- Implement custom slot types for domain-specific values

## Common Issues

- symptoms: "Bot doesn't ask for required slot value"
  diagnosis: "Slot not marked as required or missing elicitation prompt."
  resolution: "Set slot as required. Add elicitation prompt."

- symptoms: "Slot captures wrong value from utterance"
  diagnosis: "Wrong slot type or ambiguous slot position in utterance."
  resolution: "Use correct slot type. Add more utterance patterns with slot markers."

## Output Format

```yaml
root_cause: "slot_resolution — <specific_cause>"
evidence:
  - type: slot_config
    content: "<slot configuration>"
  - type: recognition_result
    content: "<slot values in response>"
severity: MEDIUM
mitigation:
  immediate: "Fix slot type or elicitation configuration"
  long_term: "Implement slot validation and testing"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or overwrite a published bot alias that is actively serving production traffic
- **NEVER** modify intent or slot configurations on a live bot version without creating a new draft version first
- **NEVER** remove Lambda fulfillment hooks from intents without verifying downstream dependencies
- **NEVER** disable conversation logs on bots subject to compliance or audit requirements
- **NEVER** migrate a V1 bot to V2 in-place without maintaining the V1 bot as a fallback

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
