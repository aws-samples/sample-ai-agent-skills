---
title: "B2 — Missed Utterances"
description: "Diagnose and reduce missed utterance rates"
status: active
severity: MEDIUM
triggers:
  - "high missed utterance rate"
  - "users not understood"
  - "fallback too frequent"
owner: devops-agent
objective: "Identify and reduce missed utterances"
context: "Missed utterances are inputs that don't match any intent above the confidence threshold. High rates indicate insufficient training data or poor intent design."
---

## Phase 1 — Triage

MUST:
- Check missed utterance metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Lex --metric-name MissedUtteranceCount --dimensions Name=BotId,Value=<bot-id> --start-time <start> --end-time <end> --period 86400 --statistics Sum`
- Review conversation logs for missed utterances (if enabled)
- Check FallbackIntent invocation count
- Verify bot is built with latest utterances

SHOULD:
- Analyze patterns in missed utterances
- Check confidence threshold setting
- Compare missed utterance rate over time

MAY:
- Export conversation logs for analysis
- Check per-intent recognition rates

## Phase 2 — Remediate

MUST:
- Add missed utterance patterns as sample utterances to appropriate intents
- Rebuild bot after adding utterances
- Monitor missed utterance rate after changes

SHOULD:
- Lower confidence threshold if too restrictive
- Create new intents for common unhandled patterns
- Implement regular utterance review cycle

MAY:
- Set up CloudWatch alarms for missed utterance rate

## Common Issues

- symptoms: "Missed utterance rate above 30%"
  diagnosis: "Insufficient sample utterances or missing intents."
  resolution: "Review missed utterances. Add as samples. Create new intents if needed."

- symptoms: "Missed utterances spike after bot update"
  diagnosis: "New version removed utterances or changed confidence threshold."
  resolution: "Compare versions. Restore missing utterances. Check threshold."

## Output Format

```yaml
root_cause: "missed_utterances — <specific_cause>"
evidence:
  - type: missed_rate
    content: "<missed utterance metrics>"
  - type: patterns
    content: "<common missed utterance patterns>"
severity: MEDIUM
mitigation:
  immediate: "Add missed patterns as sample utterances"
  long_term: "Implement continuous utterance improvement cycle"
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
