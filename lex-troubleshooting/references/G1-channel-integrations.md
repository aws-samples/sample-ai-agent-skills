---
title: "G1 — Channel Integration Issues"
description: "Diagnose Lex channel integration problems (Slack, Facebook, etc.)"
status: active
severity: MEDIUM
triggers:
  - "Slack integration not working"
  - "Facebook Messenger bot error"
  - "channel integration failed"
  - "messaging platform connection"
owner: devops-agent
objective: "Identify and resolve channel integration issues"
context: "Lex V2 supports channel integrations with Slack, Facebook Messenger, and Twilio. Each requires platform-specific configuration and credentials."
---

## Phase 1 — Triage

MUST:
- List bot channels: `aws lexv2-models list-bot-channels --bot-id <bot-id>`
- Check channel configuration: verify platform credentials are valid
- Verify bot alias used by channel is active
- Check bot locale matches channel language

SHOULD:
- Verify platform webhook URL is correctly configured
- Check platform app/bot credentials haven't expired
- Test bot directly via recognize-text to isolate channel issues

MAY:
- Check platform-side logs (Slack, Facebook developer console)
- Review channel creation CloudTrail events

## Phase 2 — Remediate

MUST:
- Update expired platform credentials
- Verify webhook URL is accessible from platform
- Ensure bot alias points to a built version

SHOULD:
- Test channel integration after any bot update
- Configure channel-specific response cards if needed

MAY:
- Implement channel health monitoring

## Common Issues

- symptoms: "Slack bot doesn't respond to messages"
  diagnosis: "Slack app credentials expired or webhook URL incorrect."
  resolution: "Update Slack app credentials. Verify webhook URL in Slack app settings."

- symptoms: "Channel shows bot as offline"
  diagnosis: "Bot alias points to failed or unbuilt version."
  resolution: "Update alias to point to a successfully built version."

## Output Format

```yaml
root_cause: "channel_integration — <specific_cause>"
evidence:
  - type: channel_config
    content: "<channel configuration>"
  - type: platform_status
    content: "<platform connection status>"
severity: MEDIUM
mitigation:
  immediate: "Fix platform credentials or webhook"
  long_term: "Implement channel health monitoring"
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
