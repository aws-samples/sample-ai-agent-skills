---
title: "B2 — Transcript Generation Failures"
description: "Diagnose Contact Lens transcript generation failures"
status: active
severity: HIGH
triggers: ["no transcript", "transcription failed", "transcript missing", "empty transcript"]
owner: devops-agent
objective: "Resolve transcript generation failures for Contact Lens contacts"
context: "Transcription is the foundation for all Contact Lens analytics. Without transcripts, sentiment, categories, and other analytics cannot be generated. Transcription depends on audio quality, language support, and proper configuration."
---

## Phase 1 — Triage

MUST:
- Verify Contact Lens is enabled (see A1)
- Check if the contact has a recording: `aws connect describe-contact --instance-id <instance-id> --contact-id <contact-id>`
- Check S3 for analytics output: `aws s3 ls s3://<connect-bucket>/connect/<instance-alias>/ContactLens/`

SHOULD:
- Verify the call recording exists in S3 (transcription requires audio)
- Check if the language is supported for transcription
- Verify the contact duration was sufficient (very short calls may not generate transcripts)

MAY:
- Check CloudWatch for Contact Lens processing errors
- Review multiple contacts to determine if the issue is systemic

## Phase 2 — Remediate

MUST:
- Ensure call recording is enabled in the contact flow (transcription requires audio)
- Verify the language in the analytics block matches the call language
- Confirm the S3 bucket has correct permissions for the Connect service

SHOULD:
- Check S3 bucket policy allows connect.amazonaws.com to write objects
- Verify the S3 bucket is in the same region as the Connect instance
- Add custom vocabulary for domain-specific terms

MAY:
- Enable server-side encryption on the S3 bucket with a KMS key the Connect service can use
- Set up S3 event notifications for new analytics files

## Safety Ratings

```
safety_ratings:
  - "describe-contact: GREEN — read-only contact details"
  - "s3 ls: GREEN — read-only bucket listing"
  - "s3 bucket policy modification: RED — affects bucket access control"
  - "contact flow modification: RED — modifies live call routing"
```

## Escalation Conditions

- No transcripts generated for any contacts despite correct configuration
- Transcripts generated for some contacts but not others with no pattern
- S3 bucket permissions correct but no analytics files appearing
- Contact Lens processing errors in CloudWatch with no clear resolution

## Data Sensitivity

- `aws connect describe-contact`: MEDIUM — returns contact metadata
- `aws s3 ls` (analytics bucket): MEDIUM — reveals analytics file structure
- Call recordings: HIGH — contain full audio of customer interactions
- Transcripts: HIGH — contain full text of customer conversations

## Prohibited Actions

- NEVER access call recordings or transcripts without authorization
- NEVER modify S3 bucket policies without understanding the impact on other services
- NEVER disable call recording to troubleshoot transcription — recording is required
- NEVER share transcripts outside authorized systems

## Phase 3 — Rollback

1. Revert S3 bucket policy changes if they caused access issues for other services
2. Revert contact flow changes if recording configuration was modified
3. Verify call recording is still functioning after any changes
4. Document the transcription failure and resolution
5. Verify transcripts are being generated for new contacts

## Common Issues

- symptoms: "Contact has recording but no transcript in S3"
  diagnosis: "Contact Lens analytics block not included in the contact flow — recording exists but analytics not triggered."
  resolution: "Add 'Set recording and analytics behavior' block with Contact Lens enabled to the contact flow."

## Output Format

```yaml
root_cause: "transcript_generation — <specific_cause>"
evidence:
  - type: transcription_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix transcription configuration"
  long_term: "Implement transcript generation monitoring"
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
