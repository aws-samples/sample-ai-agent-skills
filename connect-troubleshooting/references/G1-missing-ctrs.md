---
title: "G1 — Missing Contact Trace Records"
description: "Diagnose missing or delayed CTR data"
status: active
severity: MEDIUM
triggers:
  - "CTRs missing"
  - "No contact records"
  - "Reporting data gap"
  - "CTR delay"
owner: devops-agent
objective: "Identify why CTRs are missing and restore data completeness"
context: "CTRs are generated after each contact ends. They can take up to 24 hours to appear in Connect's built-in reporting. Kinesis streaming provides near-real-time CTR delivery. Missing CTRs may be delayed, not lost."
---

## Phase 1 — Triage

MUST:
- Check if the CTRs are delayed (wait up to 24 hours for built-in reporting)
- If using Kinesis: check stream health and consumer application
- Verify the contact actually completed (in-progress contacts don't have CTRs)
- Check the time range and filters in the reporting query

SHOULD:
- Check Kinesis stream metrics for delivery issues
- Verify the S3 bucket for CTR exports is accessible
- Check if the issue affects all contacts or specific ones

MAY:
- Search for the specific contact: `aws connect search-contacts --instance-id <id>`
- Check CloudWatch metrics for contact counts

## Phase 2 — Remediate

MUST:
- If delayed: wait for the 24-hour delivery window
- If Kinesis issue: fix stream configuration and permissions
- If S3 issue: fix bucket permissions and delivery configuration

SHOULD:
- Configure Kinesis streaming for near-real-time CTR access
- Verify the consumer application is processing records
- Check for data gaps and backfill if possible

MAY:
- Implement CTR delivery monitoring
- Set up alerts for CTR delivery delays

## Common Issues

- symptoms: "CTRs not showing for recent calls"
  diagnosis: "Normal delivery delay — CTRs can take up to 24 hours."
  resolution: "Wait 24 hours. For real-time needs, configure Kinesis streaming."

- symptoms: "Kinesis stream has no CTR records"
  diagnosis: "CTR streaming not enabled or stream configuration incorrect."
  resolution: "Enable CTR streaming in instance data streaming settings."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If agent routing or security profiles were modified, restore original profile assignments
2. If phone number associations were changed, reassociate the number with the original contact flow
3. If recording or streaming configuration was changed, restore original storage settings
4. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "missing_ctrs — <specific cause>"
evidence:
  - type: streaming_config
    content: "<CTR delivery configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix delivery configuration or wait for delay"
  long_term: "Implement Kinesis streaming and delivery monitoring"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
