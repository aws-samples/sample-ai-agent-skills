---
title: "A1 — Bot Creation Failures"
description: "Diagnose bot creation and configuration issues in Lex V2"
status: active
severity: HIGH
triggers:
  - "bot creation failed"
  - "cannot create bot"
  - "CreateBot error"
owner: devops-agent
objective: "Identify and resolve bot creation failures"
context: "Bot creation requires IAM role, locale configuration, and proper permissions. Common failures: IAM role issues, invalid locale, quota exceeded."
---

## Phase 1 — Triage

MUST:
- List bots: `aws lexv2-models list-bots`
- Check IAM role for bot: verify role has lex.amazonaws.com trust policy
- Check CloudTrail for CreateBot errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBot --max-results 10`
- Verify service quota: `aws service-quotas get-service-quota --service-code lex --quota-code L-4D2A5E60`

SHOULD:
- Verify IAM role has required permissions (polly:SynthesizeSpeech for voice)
- Check if bot name conflicts with existing bot
- Verify locale is supported

MAY:
- Check for organization SCPs blocking Lex operations
- Review account-level Lex quotas

## Phase 2 — Remediate

MUST:
- Create IAM role with lex.amazonaws.com trust policy
- Add required permissions (polly:SynthesizeSpeech for voice bots)
- Use supported locale identifier (e.g., en_US, es_ES)

SHOULD:
- Use descriptive bot names and descriptions
- Configure idle session TTL appropriately

MAY:
- Request quota increase if limit reached

## Common Issues

- symptoms: "CreateBot returns AccessDeniedException"
  diagnosis: "Calling identity lacks lex:CreateBot permission."
  resolution: "Add lex:CreateBot and lex:CreateBotLocale permissions."

- symptoms: "Bot created but locale build fails immediately"
  diagnosis: "IAM role missing polly:SynthesizeSpeech for voice-enabled bot."
  resolution: "Add Polly permissions to the bot's IAM role."

## Output Format

```yaml
root_cause: "bot_creation — <specific_cause>"
evidence:
  - type: cloudtrail_event
    content: "<CreateBot event details>"
  - type: iam_role
    content: "<role configuration>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role or permissions"
  long_term: "Create bot creation templates with validated IAM roles"
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
