---
title: "E2 — Audit Logging"
description: "Diagnose audit logging and CloudTrail issues for Payment Cryptography"
status: active
severity: HIGH
triggers: ["audit log missing", "CloudTrail not logging", "compliance audit", "logging configuration"]
owner: devops-agent
objective: "Resolve audit logging issues for Payment Cryptography operations"
context: "PCI DSS requires comprehensive audit logging of all key management and cryptographic operations. CloudTrail is the primary audit mechanism."
---

## Phase 1 — Triage

MUST:
- Verify CloudTrail is enabled: `aws cloudtrail describe-trails`
- Check CloudTrail is logging Payment Cryptography events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=payment-cryptography.amazonaws.com --max-results 5`
- Verify data events are enabled for data plane operations

SHOULD:
- Check CloudTrail log delivery to S3 bucket
- Verify CloudWatch Logs integration for real-time monitoring

MAY:
- Check CloudTrail event selectors for Payment Cryptography
- Review log retention policies

## Phase 2 — Remediate

MUST:
- Enable CloudTrail if not active
- Configure data event logging for payment-cryptography-data operations
- Ensure log delivery to S3 is functioning

SHOULD:
- Enable CloudWatch Logs integration for real-time alerting
- Set up log retention policies per PCI DSS requirements (minimum 1 year)

MAY:
- Implement centralized logging with CloudTrail organization trail
- Set up automated compliance monitoring

## Safety Ratings

```
safety_ratings:
  - "cloudtrail describe-trails: GREEN — read-only trail configuration"
  - "cloudtrail lookup-events: GREEN — read-only event search"
  - "Enable CloudTrail: YELLOW — starts logging, incurs costs"
  - "Configure data events: YELLOW — increases log volume and costs"
```

## Escalation Conditions

- CloudTrail logging gap detected during PCI DSS audit period
- Audit logs show unauthorized key management operations
- Log delivery failures create compliance gaps
- Data plane operations not being logged

## Data Sensitivity

- CloudTrail events: MEDIUM — contain API call details and principal identities
- Log delivery S3 bucket: MEDIUM — contains audit trail
- CloudWatch Logs: MEDIUM — contains real-time event data
- Event details: LOW to MEDIUM — API parameters (no key material)

## Prohibited Actions

- NEVER disable CloudTrail logging for Payment Cryptography
- NEVER delete audit logs within the PCI DSS retention period
- NEVER modify CloudTrail events or log files
- NEVER store audit logs in unencrypted S3 buckets

## Phase 3 — Rollback

1. If CloudTrail was modified: restore original trail configuration
2. If event selectors were changed: revert to previous selectors
3. If log delivery was disrupted: fix S3 bucket permissions and verify delivery
4. Verify no audit log gaps exist after changes
5. Document all logging changes for PCI DSS compliance

## Common Issues

- symptoms: "No CloudTrail events for Payment Cryptography data plane operations"
  diagnosis: "Data events are not enabled for payment-cryptography-data."
  resolution: "Enable data event logging: aws cloudtrail put-event-selectors with advanced event selectors for payment-cryptography-data."

- symptoms: "CloudTrail events missing for specific time period"
  diagnosis: "CloudTrail was temporarily disabled or S3 delivery failed."
  resolution: "Check trail status and S3 bucket permissions. Enable trail if disabled. Fix S3 delivery issues."

## Output Format

```yaml
root_cause: "audit_logging — <specific_cause>"
evidence:
  - type: logging_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix logging configuration"
  long_term: "Implement logging monitoring and alerting"
```

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
