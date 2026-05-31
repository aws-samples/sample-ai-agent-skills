---
title: "H1 — Conversation Log Issues"
description: "Diagnose conversation logging configuration and data problems"
status: active
severity: MEDIUM
triggers:
  - "conversation logs missing"
  - "logs not appearing"
  - "audio logs not saved"
  - "log configuration error"
owner: devops-agent
objective: "Identify and resolve conversation logging issues"
context: "Conversation logs capture text and audio interactions. Text logs go to CloudWatch Logs, audio logs go to S3. Requires IAM role with proper permissions. Configured per bot alias."
---

## Phase 1 — Triage

MUST:
- Check alias log settings: `aws lexv2-models describe-bot-alias --bot-id <bot-id> --bot-alias-id <alias-id> --query 'conversationLogSettings'`
- Verify IAM role exists and has correct permissions
- Check CloudWatch log group exists (for text logs)
- Check S3 bucket exists and is accessible (for audio logs)

SHOULD:
- Verify IAM role trust policy allows lexv2.amazonaws.com
- Check CloudWatch log group for recent log entries
- Verify S3 bucket policy allows Lex writes

MAY:
- Check for KMS encryption configuration on logs
- Review log retention settings

## Phase 2 — Remediate

MUST:
- Configure conversation logs on the bot alias
- Create IAM role with logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents
- Add trust policy for lexv2.amazonaws.com

SHOULD:
- Set appropriate log retention period
- Enable both text and audio logs for full visibility

MAY:
- Implement log analysis pipeline

## Common Issues

- symptoms: "Conversation logs enabled but no entries in CloudWatch"
  diagnosis: "IAM role lacks CloudWatch Logs permissions."
  resolution: "Add logs:PutLogEvents permission. Update trust policy."

- symptoms: "Audio logs not appearing in S3"
  diagnosis: "S3 bucket policy doesn't allow Lex writes or KMS key access missing."
  resolution: "Update S3 bucket policy. Grant KMS key access if encrypted."

## Output Format

```yaml
root_cause: "conversation_logs — <specific_cause>"
evidence:
  - type: log_settings
    content: "<conversation log configuration>"
  - type: iam_role
    content: "<IAM role permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM permissions or log configuration"
  long_term: "Implement log monitoring and analysis"
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
