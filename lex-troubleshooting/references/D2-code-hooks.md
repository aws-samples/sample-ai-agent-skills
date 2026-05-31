---
title: "D2 — Dialog Code Hook Issues"
description: "Diagnose dialog code hook (validation) failures"
status: active
severity: MEDIUM
triggers:
  - "code hook error"
  - "validation not working"
  - "dialog hook failed"
  - "slot validation error"
owner: devops-agent
objective: "Identify and resolve dialog code hook issues"
context: "Dialog code hooks run during slot elicitation for validation. They use the same Lambda function as fulfillment but with different invocation source. Response format controls dialog flow."
---

## Phase 1 — Triage

MUST:
- Check intent dialog code hook: `aws lexv2-models describe-intent --bot-id <bot-id> --bot-version DRAFT --locale-id en_US --intent-id <intent-id> --query 'dialogCodeHook'`
- Verify Lambda function handles DialogCodeHook invocation source
- Check Lambda logs for code hook invocations
- Test dialog flow with recognize-text

SHOULD:
- Verify Lambda response includes correct dialogAction
- Check if code hook is enabled on the intent
- Test validation logic independently

MAY:
- Review slot-level code hook configuration
- Check for code hook timeout issues

## Phase 2 — Remediate

MUST:
- Enable dialog code hook on the intent
- Handle DialogCodeHook invocation source in Lambda
- Return correct dialogAction in response (ElicitSlot, Delegate, etc.)

SHOULD:
- Implement proper validation messages for invalid slot values
- Test all dialog paths

MAY:
- Implement code hook testing framework

## Common Issues

- symptoms: "Validation not triggering during conversation"
  diagnosis: "Dialog code hook not enabled on the intent."
  resolution: "Enable dialogCodeHook on the intent. Rebuild bot."

- symptoms: "Code hook runs but doesn't re-prompt for invalid value"
  diagnosis: "Lambda response missing ElicitSlot dialogAction."
  resolution: "Return ElicitSlot action with validation message for invalid values."

## Output Format

```yaml
root_cause: "code_hook — <specific_cause>"
evidence:
  - type: hook_config
    content: "<dialog code hook configuration>"
  - type: lambda_response
    content: "<Lambda response format>"
severity: MEDIUM
mitigation:
  immediate: "Fix code hook configuration or Lambda response"
  long_term: "Implement dialog flow testing"
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
