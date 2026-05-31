---
title: "A2 — Campaign Delivery Failures"
description: "Diagnose campaign delivery issues and low delivery rates"
status: active
severity: HIGH
triggers:
  - "campaign not delivering"
  - "low delivery rate"
  - "messages not sent"
  - "campaign delivery failed"
owner: devops-agent
objective: "Identify and resolve campaign delivery failures"
context: "Campaign delivery depends on channel health, endpoint validity, opt-out status, and account limits. Common failures: opted-out endpoints, invalid addresses, channel throttling, spending limits."
---

## Phase 1 — Triage

MUST:
- Check campaign status: `aws pinpoint get-campaign --application-id <app-id> --campaign-id <campaign-id>`
- Get campaign activities: `aws pinpoint get-campaign-activities --application-id <app-id> --campaign-id <campaign-id>`
- Check delivery metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Pinpoint --metric-name DirectSendMessagePermanentFailure --dimensions Name=ApplicationId,Value=<app-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Verify channel is enabled and configured: `aws pinpoint get-email-channel --application-id <app-id>`

SHOULD:
- Check endpoint opt-out status for sample endpoints
- Verify SMS spending limit not reached
- Review bounce and complaint rates for email

MAY:
- Check Kinesis event stream for delivery events
- Review per-endpoint delivery status

## Phase 2 — Remediate

MUST:
- Fix channel configuration issues
- Remove invalid or opted-out endpoints from segment
- Increase SMS spending limit if exhausted

SHOULD:
- Implement endpoint validation before campaign launch
- Monitor delivery metrics in real-time during campaigns

MAY:
- Set up delivery failure alerts via CloudWatch

## Common Issues

- symptoms: "Campaign shows 0 deliveries but segment has endpoints"
  diagnosis: "All endpoints opted out, invalid addresses, or channel disabled."
  resolution: "Check endpoint opt-out status. Validate addresses. Verify channel enabled."

- symptoms: "SMS campaign stops mid-delivery"
  diagnosis: "SMS spending limit reached ($1.00 default)."
  resolution: "Request spending limit increase via AWS Support."

## Output Format

```yaml
root_cause: "campaign_delivery — <specific_cause>"
evidence:
  - type: campaign_activity
    content: "<campaign activity details>"
  - type: delivery_metrics
    content: "<CloudWatch delivery metrics>"
severity: HIGH
mitigation:
  immediate: "Fix delivery blockers"
  long_term: "Implement pre-campaign validation and monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — campaign delivery sends messages to customers
- Service Disruption Risk: HIGH — delivery failures affect customer communication
- Compliance Impact: HIGH — message delivery must comply with all messaging regulations

## Escalation Conditions
- Campaign delivery completely fails
- Messages delivered to wrong recipients
- Delivery rate significantly below expectations
- Campaign sends duplicate messages
- Campaign exceeds sending limits

## Data Sensitivity
- Recipient data — RESTRICTED (PII)
- Message content — INTERNAL
- Delivery metrics — INTERNAL
- Bounce and complaint data — INTERNAL
- Never expose recipient PII in delivery logs

## Prohibited Actions
- Do NOT send messages to customers who have opted out
- Do NOT exceed regulatory sending limits
- Do NOT ignore delivery failure patterns
- Do NOT send messages during blackout periods
- Do NOT bypass message content review
- Do NOT share delivery reports containing PII

## Phase 3 — Rollback
- Pause or cancel active campaigns with delivery issues
- Restore previous campaign configurations
- Revert sending configuration changes
- Verify delivery metrics after changes
- Process opt-out requests from incorrectly targeted recipients
- Document delivery issues and compliance impact

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
