---
title: "E1 — Push Notification Configuration"
description: "Diagnose push notification channel configuration issues"
status: active
severity: HIGH
triggers:
  - "push notifications not working"
  - "APNs configuration error"
  - "FCM setup failed"
  - "push channel error"
owner: devops-agent
objective: "Identify and resolve push notification configuration issues"
context: "Push notifications require platform-specific credentials: APNs signing key/certificate for iOS, FCM server key for Android, ADM credentials for Kindle. Each platform configured separately."
---

## Phase 1 — Triage

MUST:
- Check APNs channel: `aws pinpoint get-apns-channel --application-id <app-id>`
- Check GCM/FCM channel: `aws pinpoint get-gcm-channel --application-id <app-id>`
- Verify channel is enabled and credentials are set
- Check CloudTrail for UpdateApnsChannel/UpdateGcmChannel errors

SHOULD:
- Verify APNs certificate/key hasn't expired
- Confirm FCM server key is valid
- Check if using APNs sandbox vs production environment

MAY:
- Check ADM channel for Kindle: `aws pinpoint get-adm-channel --application-id <app-id>`
- Verify Baidu channel if targeting China

## Phase 2 — Remediate

MUST:
- Update expired APNs certificates/keys
- Configure correct FCM server key or service account
- Match APNs environment (sandbox/production) to app build

SHOULD:
- Use APNs token-based authentication (signing key) over certificate
- Test push delivery to a single device first

MAY:
- Set up push notification delivery monitoring

## Common Issues

- symptoms: "APNs channel enabled but iOS push not delivered"
  diagnosis: "APNs certificate expired or wrong environment (sandbox vs production)."
  resolution: "Update certificate. Match environment to app build type."

- symptoms: "FCM channel returns InvalidRegistration"
  diagnosis: "Device token is invalid or app uninstalled."
  resolution: "Clean up invalid device tokens. Update endpoint with fresh token."

## Output Format

```yaml
root_cause: "push_config — <specific_cause>"
evidence:
  - type: apns_channel
    content: "<APNs channel configuration>"
  - type: gcm_channel
    content: "<FCM channel configuration>"
severity: HIGH
mitigation:
  immediate: "Update platform credentials"
  long_term: "Implement credential rotation and monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — push configuration involves platform credentials
- Service Disruption Risk: MEDIUM — configuration issues prevent push delivery
- Compliance Impact: MEDIUM — push notifications must respect user preferences

## Escalation Conditions
- Push notification channel completely non-functional
- APNs or FCM credentials expired or revoked
- Push configuration changes break delivery
- Platform-specific configuration issues
- Push channel quota exceeded

## Data Sensitivity
- APNs certificates and keys — RESTRICTED
- FCM server keys — RESTRICTED
- Push channel configurations — INTERNAL
- Device token management — INTERNAL
- Never expose platform credentials

## Prohibited Actions
- Do NOT share APNs or FCM credentials
- Do NOT store platform credentials in plaintext
- Do NOT bypass push notification opt-in requirements
- Do NOT ignore credential expiration warnings
- Do NOT configure push channels without testing
- Do NOT expose push configuration details externally

## Phase 3 — Rollback
- Restore previous push channel configurations
- Re-upload APNs certificates if expired
- Restore FCM credentials
- Verify push delivery after configuration changes
- Restore device token management settings
- Document push configuration issues and resolutions

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
