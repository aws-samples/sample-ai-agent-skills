---
title: "G1 — CloudTrail Organization Trail Issues"
description: "Diagnose issues with the Control Tower organization-level CloudTrail trail"
status: active
severity: HIGH
triggers:
  - "CloudTrail not logging"
  - "Organization trail error"
  - "Missing audit logs"
  - "Trail delivery failure"
owner: devops-agent
objective: "Restore CloudTrail organization trail functionality"
context: "Control Tower creates an organization-level CloudTrail trail that logs API events from all member accounts to the Log Archive account's S3 bucket. Issues include delivery failures, S3 bucket policy problems, KMS key issues, or trail modifications causing drift."
---

## Phase 1 — Triage

MUST:
- Check trail status: `aws cloudtrail describe-trails` and `aws cloudtrail get-trail-status --name <trail-name>`
- Verify the trail is an organization trail and is logging
- Check S3 bucket delivery status (last delivery time, last delivery error)
- Verify the Log Archive account S3 bucket exists and has correct policy

SHOULD:
- Check if the trail was modified outside of CT (causes drift)
- Verify KMS encryption key is accessible
- Check CloudWatch Logs delivery if configured

MAY:
- Check CloudTrail for trail modification events
- Verify the trail configuration matches CT expectations

## Phase 2 — Remediate

MUST:
- If trail modified: re-register the Security OU or repair landing zone to restore
- If S3 bucket policy issue: verify the bucket policy allows CloudTrail delivery from the organization
- If KMS key issue: verify key policy allows CloudTrail to encrypt/decrypt

SHOULD:
- Do NOT modify the CT-managed trail directly
- Verify log delivery resumes after fix
- Check for gaps in log coverage

MAY:
- Set up CloudWatch alarms for trail delivery failures
- Implement S3 bucket policy monitoring

## Common Issues

- symptoms: "CloudTrail shows trail is not logging"
  diagnosis: "Trail was stopped or modified outside of Control Tower."
  resolution: "Repair the landing zone or re-register the Security OU to restore the trail."

- symptoms: "S3 delivery errors"
  diagnosis: "Log Archive S3 bucket policy was modified or bucket was deleted."
  resolution: "Restore the bucket policy. If bucket deleted, this is critical drift — repair landing zone."

## Safety Ratings

```
safety_ratings:
  - "describe-trails: GREEN — read-only trail configuration"
  - "get-trail-status: GREEN — read-only trail status"
  - "re-register Security OU to restore trail: YELLOW — recoverable, resets CT baselines"
  - "repair landing zone: RED — modifies foundational governance structure"
  - "modify S3 bucket policy in Log Archive: RED — can break log delivery for all accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Fix requires moving the Audit or Log Archive account
- Organization trail was deleted (not just stopped)
- Log Archive S3 bucket was deleted
- Drift affects multiple OUs or accounts

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- HIGH: CloudTrail logs (contain all API activity across all accounts)
- MEDIUM: OU structure, control enablement status
- MEDIUM: S3 bucket policies and KMS key policies

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- Do NOT modify the CT-managed trail directly — use CT repair mechanisms
- If the S3 bucket policy was modified, use landing zone repair to restore — do NOT manually edit the policy
- Document the trail configuration and S3 bucket policy before any remediation
- If there are gaps in log coverage, document the time range for compliance and audit purposes

## Output Format

```yaml
root_cause: "cloudtrail — <specific issue>"
evidence:
  - type: cloudtrail_status
    content: "<trail status and delivery details>"
severity: HIGH
mitigation:
  immediate: "Restore trail via CT repair mechanisms"
  long_term: "Monitor trail health and protect CT-managed resources"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
