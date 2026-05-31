---
title: "C1 — Platform Application Issues"
description: "Diagnose mobile push platform application configuration problems"
status: active
severity: HIGH
triggers:
  - "platform application"
  - "APNs"
  - "FCM"
  - "GCM"
  - "push platform"
owner: devops-agent
objective: "Resolve mobile push platform application configuration and credential issues"
context: "Platform applications connect SNS to mobile push services (APNs for iOS, FCM for Android). They require platform-specific credentials that must be kept current. Expired or invalid credentials cause all push notifications to fail."
---

## Phase 1 — Triage

MUST:
- List platform applications: `aws sns list-platform-applications`
- Get application attributes: `aws sns get-platform-application-attributes --platform-application-arn <app-arn>`
- Check for enabled status and credential validity
- Check CloudWatch metrics for push failures

SHOULD:
- Verify APNs certificate expiration (iOS)
- Verify FCM server key or service account validity (Android)
- Check for platform-specific error messages in delivery logs

MAY:
- Test with a known-good device token
- Review platform application event notifications

## Phase 2 — Remediate

MUST:
- If APNs cert expired: upload new certificate or switch to token-based auth
- If FCM key invalid: update server key or service account JSON
- If platform wrong: verify APNs sandbox vs production environment

SHOULD:
- Use token-based auth for APNs (no certificate expiration)
- Set up credential expiration monitoring
- Enable platform application event notifications

MAY:
- Implement automated credential rotation
- Use Amazon Pinpoint for advanced push management

## Common Issues

- symptoms: "All iOS push notifications failing"
  diagnosis: "APNs certificate expired or using sandbox cert in production."
  resolution: "Upload new APNs certificate. Verify sandbox vs production environment."

- symptoms: "All Android push notifications failing"
  diagnosis: "FCM server key invalid or revoked."
  resolution: "Update FCM credentials. Use service account JSON for FCM v1 API."

## Output Format

```yaml
root_cause: "platform_app — <specific_cause>"
evidence:
  - type: app_attributes
    content: "<platform application configuration>"
  - type: credentials
    content: "<credential status>"
severity: HIGH
mitigation:
  immediate: "Update platform credentials"
  long_term: "Implement credential monitoring and rotation"
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
