---
title: "E2 — Push Notification Delivery Failures"
description: "Diagnose push notification delivery issues"
status: active
severity: MEDIUM
triggers:
  - "push not received"
  - "push delivery failed"
  - "device not receiving notifications"
  - "push token invalid"
owner: devops-agent
objective: "Identify and resolve push notification delivery failures"
context: "Push delivery failures stem from invalid device tokens, platform service outages, payload size limits, or endpoint opt-out. Platform feedback (APNs/FCM) indicates specific failure reasons."
---

## Phase 1 — Triage

MUST:
- Check endpoint details: `aws pinpoint get-endpoint --application-id <app-id> --endpoint-id <endpoint-id>`
- Verify device token is current and valid
- Check push delivery metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Pinpoint --metric-name DirectSendMessagePermanentFailure --dimensions Name=ApplicationId,Value=<app-id> Name=Channel,Value=GCM --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Verify endpoint opt-out status

SHOULD:
- Check payload size (APNs: 4KB, FCM: 4KB limit)
- Verify endpoint channel type matches push platform
- Test with direct message send to single endpoint

MAY:
- Check platform service status (APNs, FCM)
- Review event stream for delivery events

## Phase 2 — Remediate

MUST:
- Update stale device tokens
- Remove endpoints with permanently invalid tokens
- Ensure payload within size limits

SHOULD:
- Implement token refresh mechanism in mobile app
- Handle platform feedback to clean invalid tokens

MAY:
- Set up delivery failure alerting

## Common Issues

- symptoms: "Push sent successfully but device doesn't receive"
  diagnosis: "Device token expired or app uninstalled."
  resolution: "Implement token refresh. Clean stale tokens from endpoints."

- symptoms: "Push delivery returns PayloadTooLarge"
  diagnosis: "Message payload exceeds 4KB platform limit."
  resolution: "Reduce payload size. Use data-only messages with client-side rendering."

## Output Format

```yaml
root_cause: "push_delivery — <specific_cause>"
evidence:
  - type: endpoint_details
    content: "<endpoint configuration>"
  - type: delivery_metrics
    content: "<push delivery metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix device tokens or payload size"
  long_term: "Implement token lifecycle management"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — push notifications may contain personalized content
- Service Disruption Risk: MEDIUM — delivery failures affect user engagement
- Compliance Impact: MEDIUM — push notifications must respect user preferences

## Escalation Conditions
- Push notification delivery rate drops significantly
- Push notifications delivered to wrong user segments
- Push notification content contains errors
- Platform-specific delivery failures (APNs/FCM)
- Push notification delivery latency exceeds thresholds

## Data Sensitivity
- Device tokens — RESTRICTED
- Notification content — INTERNAL
- User preferences — CONFIDENTIAL
- Delivery metrics — INTERNAL
- Never expose device tokens or user identifiers

## Prohibited Actions
- Do NOT send push notifications to users who have opted out
- Do NOT expose device tokens in logs
- Do NOT bypass push notification frequency limits
- Do NOT send notifications with misleading content
- Do NOT ignore delivery failure patterns
- Do NOT share push notification credentials (APNs/FCM keys)

## Phase 3 — Rollback
- Restore previous push notification configurations
- Revert APNs/FCM credential changes
- Restore notification templates
- Verify push delivery after changes
- Restore user preference configurations
- Document push delivery issues and resolutions

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
