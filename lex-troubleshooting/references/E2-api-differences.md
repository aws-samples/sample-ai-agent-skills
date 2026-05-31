---
title: "E2 — V1 vs V2 API Differences"
description: "Diagnose issues caused by V1/V2 API confusion"
status: active
severity: MEDIUM
triggers:
  - "wrong API version"
  - "bot not found"
  - "API mismatch"
  - "lex-models vs lexv2-models"
owner: devops-agent
objective: "Identify and resolve V1/V2 API confusion issues"
context: "V1 (lex-models, lex-runtime) and V2 (lexv2-models, lexv2-runtime) are completely separate APIs with different identifiers, endpoints, and data models."
---

## Phase 1 — Triage

MUST:
- Identify which API version the bot was created with
- V1 check: `aws lex-models get-bots`
- V2 check: `aws lexv2-models list-bots`
- Verify client application uses matching API version

SHOULD:
- Check if bot-id (V2) vs bot-name (V1) is being used correctly
- Verify runtime API matches model API version
- Check SDK version supports V2 APIs

MAY:
- Review application code for API version references
- Check for mixed V1/V2 usage in infrastructure code

## Phase 2 — Remediate

MUST:
- Use lexv2-models/lexv2-runtime for V2 bots
- Use lex-models/lex-runtime for V1 bots (legacy only)
- Update client code to use correct API version

SHOULD:
- Migrate all V1 bots to V2
- Update SDK to latest version supporting V2

MAY:
- Add API version validation in deployment pipeline

## Common Issues

- symptoms: "ResourceNotFoundException when accessing bot"
  diagnosis: "Using V1 API with V2 bot ID or vice versa."
  resolution: "Use correct API version. V2 bots use lexv2-models."

- symptoms: "Runtime API returns unexpected response format"
  diagnosis: "Using V1 runtime with V2 bot or vice versa."
  resolution: "Match runtime API version to bot version."

## Output Format

```yaml
root_cause: "api_differences — <specific_cause>"
evidence:
  - type: api_version
    content: "<API version being used>"
  - type: bot_type
    content: "<V1 or V2 bot>"
severity: MEDIUM
mitigation:
  immediate: "Switch to correct API version"
  long_term: "Standardize on V2 API across all applications"
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
