---
title: "E4 — S3 Call Recording Issues"
description: "Diagnose issues with call recording storage in S3"
status: active
severity: MEDIUM
triggers:
  - "Recordings missing"
  - "Cannot play recording"
  - "S3 recording error"
  - "Recording not saved"
owner: devops-agent
objective: "Restore call recording functionality"
context: "Connect stores call recordings in S3. Recording must be enabled in the contact flow via 'Set recording and analytics behavior' block. Recordings are encrypted with KMS. Issues include: recording not enabled, S3 bucket permissions, KMS key access, or storage configuration."
---

## Phase 1 — Triage

MUST:
- Verify recording is enabled in the contact flow ("Set recording and analytics behavior" block)
- Check the S3 bucket configuration for the instance
- Verify the S3 bucket exists and Connect has write permissions
- Check KMS key permissions if encryption is configured

SHOULD:
- Verify the recording is set for the correct participant (agent, customer, or both)
- Check if the recording block is placed before the transfer to queue
- Verify S3 bucket lifecycle rules aren't deleting recordings prematurely

MAY:
- Check CloudTrail for S3 access denied events
- Verify the recording appears in the CTR after the 24-hour delay

## Phase 2 — Remediate

MUST:
- Add "Set recording and analytics behavior" block in the contact flow before queue transfer
- Fix S3 bucket permissions for Connect
- Fix KMS key policy if encryption is blocking

SHOULD:
- Verify recordings appear in the Connect console after fix
- Check S3 bucket for the recording files
- Test recording playback

MAY:
- Implement S3 lifecycle policies for recording retention
- Set up monitoring for recording failures

## Common Issues

- symptoms: "No recordings for any calls"
  diagnosis: "Recording not enabled in the contact flow."
  resolution: "Add 'Set recording and analytics behavior' block before the queue transfer."

- symptoms: "Recording exists but cannot be played"
  diagnosis: "KMS key permissions don't allow decryption."
  resolution: "Update KMS key policy to allow the user/role to decrypt."

- symptoms: "Recordings missing for some calls"
  diagnosis: "Recording block is after the queue transfer or only on some flow branches."
  resolution: "Place the recording block before any transfer and on all flow branches."


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

1. If a contact flow was modified, publish the previous saved version of the flow from the Connect console
2. If queue configuration was changed, restore original routing profile assignments and queue settings
3. If agent routing or security profiles were modified, restore original profile assignments
4. If recording or streaming configuration was changed, restore original storage settings
5. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "s3_recording — <specific cause>"
evidence:
  - type: contact_flow
    content: "<recording block configuration>"
  - type: s3_bucket
    content: "<bucket permissions and status>"
severity: MEDIUM
mitigation:
  immediate: "Fix recording configuration"
  long_term: "Implement recording monitoring and retention policies"
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
