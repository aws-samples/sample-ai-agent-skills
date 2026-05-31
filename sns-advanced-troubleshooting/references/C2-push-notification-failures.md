---
title: "C2 — Push Notification Delivery Failures"
description: "Diagnose mobile push notification delivery problems"
status: active
severity: HIGH
triggers:
  - "push notification failed"
  - "push not delivered"
  - "device token"
  - "endpoint disabled"
owner: devops-agent
objective: "Resolve mobile push notification delivery failures"
context: "Push notification failures can be caused by invalid device tokens, disabled endpoints, platform credential issues, or payload format errors. SNS disables endpoints when the platform reports the token as invalid."
---

## Phase 1 — Triage

MUST:
- Check endpoint attributes: `aws sns get-endpoint-attributes --endpoint-arn <endpoint-arn>`
- Verify endpoint is enabled (Enabled: true)
- Check device token validity
- Check delivery status logs for push failures

SHOULD:
- Verify platform application credentials are valid
- Check push payload format and size (4 KB for APNs, 4 KB for FCM)
- Review endpoint creation date (old tokens may be invalid)

MAY:
- Test with platform-specific push testing tools
- Check for device-level notification settings

## Phase 2 — Remediate

MUST:
- If endpoint disabled: re-enable after verifying token is valid: `aws sns set-endpoint-attributes --endpoint-arn <endpoint-arn> --attributes Enabled=true`
- If token invalid: update with new token from the device
- If payload too large: reduce payload size (4 KB limit)

SHOULD:
- Implement token refresh logic in the mobile app
- Handle endpoint disabled events via platform application events
- Use platform-specific payload format (APNs JSON, FCM JSON)

MAY:
- Implement endpoint cleanup for stale tokens
- Use Amazon Pinpoint for advanced push analytics

## Common Issues

- symptoms: "Push notification not received on device"
  diagnosis: "Endpoint disabled by SNS after platform reported invalid token."
  resolution: "Get new device token. Update or recreate the endpoint."

- symptoms: "Push works for some devices but not others"
  diagnosis: "Some device tokens are stale or endpoints are disabled."
  resolution: "Check endpoint enabled status. Implement token refresh in the app."

## Output Format

```yaml
root_cause: "push_failure — <specific_cause>"
evidence:
  - type: endpoint_status
    content: "<endpoint enabled and token>"
  - type: delivery_logs
    content: "<push delivery failure reason>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint or update device token"
  long_term: "Implement token refresh and endpoint lifecycle management"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Message payloads may contain sensitive application data
- Subscription endpoints reveal integration architecture
- SMS messages may contain customer PII or authentication codes
- Classification: CONFIDENTIAL - message content and subscriber endpoint data

## Prohibited Actions

- NEVER delete SNS topics with active subscriptions in production
- NEVER modify filter policies on production subscriptions without testing
- NEVER change access policies that could disrupt cross-account message delivery
- NEVER purge dead-letter queues without analyzing failed messages

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
