---
title: "H2 — KMS Audit Logging Issues"
description: "Diagnose CloudTrail logging gaps for KMS operations"
status: active
severity: MEDIUM
triggers:
  - "KMS audit"
  - "CloudTrail KMS"
  - "KMS logging"
  - "who used the key"
  - "key usage audit"
owner: devops-agent
objective: "Ensure comprehensive audit logging for all KMS operations"
context: "All KMS API calls are logged in CloudTrail. This includes cryptographic operations (Encrypt, Decrypt, GenerateDataKey), management operations (CreateKey, PutKeyPolicy), and grant operations. CloudTrail logs include the encryption context, which is valuable for auditing. AWS-managed key usage is also logged."
---

## Phase 1 — Triage

MUST:
- Verify CloudTrail is enabled: `aws cloudtrail describe-trails`
- Check for KMS events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20`
- Check for specific key usage: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::KMS::Key --max-results 20`
- Verify CloudTrail is logging management and data events

SHOULD:
- Check if CloudTrail data events are enabled for KMS (required for cryptographic operations in some configurations)
- Review CloudTrail log file validation for integrity
- Check S3 bucket for CloudTrail log delivery

MAY:
- Set up CloudWatch Logs integration for real-time alerting
- Check for CloudTrail organization trails

## Phase 2 — Remediate

MUST:
- Enable CloudTrail if not already enabled
- Ensure the trail covers all regions for comprehensive KMS auditing
- Verify CloudTrail log delivery to S3 is working

SHOULD:
- Enable CloudTrail Insights for anomaly detection on KMS usage
- Set up CloudWatch metric filters for sensitive KMS operations (DisableKey, ScheduleKeyDeletion, PutKeyPolicy)
- Enable log file validation for tamper detection

MAY:
- Create Athena queries for KMS usage analysis
- Set up SNS notifications for critical KMS events

## Common Issues

- symptoms: "Cannot find KMS decrypt events in CloudTrail"
  diagnosis: "CloudTrail may not be logging data events, or the lookup time range is too narrow."
  resolution: "Expand the time range. Check if data events are enabled for KMS."

- symptoms: "KMS events missing encryption context"
  diagnosis: "The caller did not provide encryption context during the operation."
  resolution: "Encryption context is optional but recommended. Implement it for better auditability."

## Output Format

```yaml
root_cause: "audit_logging_issue — <specific_cause>"
evidence:
  - type: cloudtrail
    content: "<CloudTrail configuration and status>"
  - type: kms_events
    content: "<sample KMS events>"
severity: MEDIUM
mitigation:
  immediate: "Enable CloudTrail and verify KMS event logging"
  long_term: "Set up alerting for sensitive KMS operations"
```


## Safety Ratings

- `describe-trails`: GREEN — Read-only CloudTrail configuration check
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis
- `describe-key`: GREEN — Read-only key metadata retrieval
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `put-event-selectors` (CloudTrail): YELLOW — Changes which events are logged; does not affect KMS operations

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Audit logs reveal unauthorized KMS API calls
- Compliance audit requires immediate log review

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- HIGH: "CloudTrail logs contain KMS API call details including principal ARNs, key IDs, and encryption context"
- MEDIUM: "Key metadata reveals encryption strategy"

## Prohibited Actions

- NEVER suggest `"kms:*"` in a key policy
- NEVER suggest making a key policy public with `"Principal": "*"`
- NEVER suggest disabling key rotation as a fix
- NEVER suggest scheduling key deletion without confirming no active usage
- NEVER suggest creating a new key to bypass policy issues (data becomes inaccessible)

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
  - command: "get-key-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-key"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-grants"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest kms:* on key policies"
  - "NEVER suggest disabling key rotation"
  - "NEVER suggest making key policies public"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
