---
title: "G3 — Log Archive Account Problems"
description: "Diagnose issues with the Control Tower Log Archive account"
status: active
severity: HIGH
triggers:
  - "Log archive"
  - "Centralized logging broken"
  - "S3 bucket access denied"
  - "Logs not delivered"
owner: devops-agent
objective: "Restore centralized logging to the Log Archive account"
context: "The Log Archive account is a CT-managed shared account that stores CloudTrail logs and Config snapshots from all governed accounts. It has specific S3 bucket policies and lifecycle rules. Modifications cause drift."
---

## Phase 1 — Triage

MUST:
- Verify the Log Archive account is in the Security OU
- Check S3 bucket existence and policies in the Log Archive account
- Verify CloudTrail is delivering to the correct bucket
- Check if the Log Archive account was modified outside CT

SHOULD:
- Verify S3 bucket lifecycle rules are intact
- Check bucket versioning and encryption settings
- Verify cross-account access policies

MAY:
- Check S3 access logs for denied requests
- Review the bucket policy for completeness

## Phase 2 — Remediate

MUST:
- If account moved: move it back to the Security OU
- If bucket policy modified: repair the landing zone to restore
- If bucket deleted: this is critical — contact AWS Support and repair landing zone

SHOULD:
- Do NOT manually modify Log Archive S3 buckets or policies
- Verify log delivery resumes after fix
- Check for log gaps during the outage

MAY:
- Implement S3 bucket policy monitoring
- Set up alerts for bucket modifications

## Common Issues

- symptoms: "CloudTrail delivery failing to Log Archive"
  diagnosis: "S3 bucket policy was modified or bucket was deleted."
  resolution: "Repair the landing zone to restore the bucket policy."

- symptoms: "Log Archive account moved out of Security OU"
  diagnosis: "Account drift — the account was moved to a different OU."
  resolution: "Move the account back to the Security OU and repair the landing zone."

## Safety Ratings

```
safety_ratings:
  - "list-buckets: GREEN — read-only bucket listing"
  - "get-bucket-policy: GREEN — read-only bucket policy"
  - "move Log Archive account back to Security OU: YELLOW — recoverable, account can be moved again"
  - "repair landing zone to restore bucket policy: RED — modifies foundational governance structure"
  - "modify Log Archive S3 bucket or policy directly: RED — can break centralized logging for all accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Fix requires moving the Audit or Log Archive account
- Log Archive S3 bucket was deleted
- Drift affects multiple OUs or accounts
- Log delivery has been broken for an extended period (compliance risk)

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- HIGH: CloudTrail and Config logs stored in Log Archive (all API activity and resource configs)
- MEDIUM: OU structure, control enablement status
- MEDIUM: S3 bucket policies and lifecycle rules

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- Do NOT manually modify Log Archive S3 buckets or policies — use CT repair mechanisms
- If the Log Archive account was moved back to the Security OU, verify log delivery resumes
- Document the bucket policy and lifecycle rules before any remediation
- If the S3 bucket was deleted, this is critical — contact AWS Support immediately

## Output Format

```yaml
root_cause: "log_archive — <specific issue>"
evidence:
  - type: s3_bucket
    content: "<bucket status and policy>"
severity: HIGH
mitigation:
  immediate: "Restore Log Archive configuration via CT repair"
  long_term: "Protect Log Archive resources from manual modification"
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
