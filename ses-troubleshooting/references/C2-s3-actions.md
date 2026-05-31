---
title: "C2 — S3 Actions"
description: "Diagnose receipt rule S3 action failures"
status: active
severity: MEDIUM
triggers:
  - "S3 action failed"
  - "email not saved to S3"
  - "receipt rule S3 error"
  - "bucket permission denied"
owner: devops-agent
objective: "Resolve S3 action failures in SES receipt rules"
context: "Receipt rule S3 actions save incoming emails to an S3 bucket. The bucket policy must grant SES permission to put objects. The bucket must be in the same region as the receipt rule. KMS encryption requires additional permissions."
---

## Phase 1 — Triage

MUST:
- Check receipt rule S3 action configuration: `aws ses describe-receipt-rule --rule-set-name <name> --rule-name <rule>`
- Verify S3 bucket exists and is in the correct region: `aws s3api get-bucket-location --bucket <bucket>`
- Check bucket policy grants SES access: `aws s3api get-bucket-policy --bucket <bucket>`
- Verify KMS key permissions if encryption is configured

SHOULD:
- Check S3 bucket for recent objects to confirm delivery
- Review CloudTrail for S3 PutObject failures from SES
- Verify the object key prefix is correct

MAY:
- Check S3 bucket versioning and lifecycle policies
- Verify bucket is not blocked by S3 Block Public Access (SES uses service principal, not public access)

## Phase 2 — Remediate

MUST:
- Add bucket policy allowing SES to put objects:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Service": "ses.amazonaws.com"},
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::<bucket>/*",
    "Condition": {"StringEquals": {"AWS:SourceAccount": "<account-id>"}}
  }
  ```
- Ensure bucket is in the same region as the receipt rule

SHOULD:
- Add KMS key policy for SES if using encryption
- Configure SNS notification on the S3 action for monitoring

MAY:
- Set up S3 lifecycle rules for email retention management
- Enable S3 event notifications for downstream processing

## Common Issues

- symptoms: "S3 action silently fails"
  diagnosis: "Bucket policy does not grant ses.amazonaws.com PutObject permission."
  resolution: "Add SES service principal to bucket policy with PutObject permission."

- symptoms: "Access denied with KMS encryption"
  diagnosis: "KMS key policy does not grant SES permission to generate data keys."
  resolution: "Add ses.amazonaws.com to KMS key policy with kms:GenerateDataKey permission."

- symptoms: "Emails saved but in wrong location"
  diagnosis: "Object key prefix misconfigured in the receipt rule action."
  resolution: "Update the S3 action object key prefix in the receipt rule."

## Output Format

```yaml
root_cause: "s3_action — <specific_cause>"
evidence:
  - type: rule_config
    content: "<receipt rule S3 action configuration>"
  - type: bucket_policy
    content: "<S3 bucket policy>"
severity: MEDIUM
mitigation:
  immediate: "Fix bucket policy, KMS permissions, or action configuration"
  long_term: "Implement monitoring for S3 action success/failure"
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

- Email content may contain customer PII and business communications
- Recipient lists contain personal email addresses
- Bounce and complaint data reveals delivery issues with specific addresses
- Classification: HIGHLY CONFIDENTIAL - email content and recipient PII

## Prohibited Actions

- NEVER send test emails to real customer addresses
- NEVER modify production DKIM/SPF/DMARC records without email team approval
- NEVER disable bounce or complaint handling in production
- NEVER remove addresses from the suppression list without investigating the original cause

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
  - "NEVER suggest disabling DKIM/SPF/DMARC"
  - "NEVER suggest removing sending limits"
  - "NEVER suggest disabling bounce/complaint handling"
