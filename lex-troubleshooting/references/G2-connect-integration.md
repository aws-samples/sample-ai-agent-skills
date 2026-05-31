---
title: "G2 — Amazon Connect Integration Issues"
description: "Diagnose Lex integration with Amazon Connect"
status: active
severity: HIGH
triggers:
  - "Lex bot not working in Connect"
  - "Connect contact flow Lex error"
  - "bot not responding in call"
  - "Connect Lex integration"
owner: devops-agent
objective: "Identify and resolve Lex-Connect integration issues"
context: "Lex bots integrate with Connect via contact flows. Requires bot association with Connect instance, proper IAM permissions, and matching locale. Voice bots need audio-capable configuration."
---

## Phase 1 — Triage

MUST:
- Check Connect-Lex association: `aws connect list-bots --instance-id <instance-id> --lex-version V2`
- Verify bot alias is associated with Connect instance
- Check bot locale matches Connect language
- Verify bot is built and alias points to valid version

SHOULD:
- Test bot independently via recognize-utterance
- Check Connect contact flow for correct bot block configuration
- Verify Connect instance has Lex permissions

MAY:
- Review Connect contact flow logs
- Check for Connect service-linked role permissions

## Phase 2 — Remediate

MUST:
- Associate bot with Connect instance: `aws connect associate-bot --instance-id <instance-id> --lex-v2-bot AliasArn=<alias-arn>`
- Ensure bot locale matches Connect contact flow language
- Build bot with voice-capable configuration

SHOULD:
- Test end-to-end with a test phone call
- Configure session attributes passing between Connect and Lex

MAY:
- Implement Connect contact flow testing automation

## Common Issues

- symptoms: "Contact flow shows 'Error' branch for Lex block"
  diagnosis: "Bot not associated with Connect instance or alias invalid."
  resolution: "Associate bot with Connect instance. Verify alias is active."

- symptoms: "Bot works in console but not in Connect call"
  diagnosis: "Locale mismatch or voice configuration issue."
  resolution: "Match bot locale to Connect language. Verify voice output configured."

## Output Format

```yaml
root_cause: "connect_integration — <specific_cause>"
evidence:
  - type: bot_association
    content: "<Connect-Lex association status>"
  - type: contact_flow
    content: "<contact flow configuration>"
severity: HIGH
mitigation:
  immediate: "Fix bot association or locale configuration"
  long_term: "Implement Connect-Lex integration testing"
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
