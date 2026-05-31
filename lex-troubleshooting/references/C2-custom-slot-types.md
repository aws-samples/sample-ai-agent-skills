---
title: "C2 — Custom Slot Type Issues"
description: "Diagnose custom slot type creation and resolution problems"
status: active
severity: MEDIUM
triggers:
  - "custom slot type not matching"
  - "slot type creation failed"
  - "slot values not recognized"
owner: devops-agent
objective: "Identify and resolve custom slot type issues"
context: "Custom slot types define domain-specific value sets. They are locale-specific and must be rebuilt after changes. Resolution strategy (TopResolution vs OriginalValue) affects behavior."
---

## Phase 1 — Triage

MUST:
- List slot types: `aws lexv2-models list-slot-types --bot-id <bot-id> --bot-version DRAFT --locale-id en_US`
- Check slot type: `aws lexv2-models describe-slot-type --bot-id <bot-id> --bot-version DRAFT --locale-id en_US --slot-type-id <slot-type-id>`
- Verify slot type values include expected inputs
- Check value resolution strategy

SHOULD:
- Add synonyms for slot type values
- Verify slot type is used by the correct slot
- Check if bot was rebuilt after slot type changes

MAY:
- Test slot type resolution with various inputs
- Check for slot type value limits

## Phase 2 — Remediate

MUST:
- Add missing values and synonyms to custom slot type
- Rebuild bot after slot type changes
- Use appropriate resolution strategy

SHOULD:
- Use OriginalValue strategy when exact user input matters
- Use TopResolution when canonical values are needed

MAY:
- Implement slot type value management automation

## Common Issues

- symptoms: "Custom slot type doesn't match user input"
  diagnosis: "Value not in slot type or missing synonyms."
  resolution: "Add value and synonyms. Rebuild bot."

- symptoms: "Slot resolves to wrong canonical value"
  diagnosis: "Synonym mapped to wrong value or resolution strategy incorrect."
  resolution: "Fix synonym mappings. Check resolution strategy."

## Output Format

```yaml
root_cause: "custom_slot_type — <specific_cause>"
evidence:
  - type: slot_type_config
    content: "<slot type values and synonyms>"
  - type: resolution_strategy
    content: "<resolution strategy>"
severity: MEDIUM
mitigation:
  immediate: "Fix slot type values and rebuild"
  long_term: "Maintain slot type value catalog"
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
