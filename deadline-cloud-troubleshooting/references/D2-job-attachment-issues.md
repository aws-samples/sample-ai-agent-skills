---
title: "D2 — Job Attachment Issues"
description: "Diagnose job attachment upload, download, and S3 access issues"
status: active
severity: HIGH
triggers: ["attachment upload failed", "attachment download error", "S3 access denied", "job files missing"]
owner: devops-agent
objective: "Resolve job attachment transfer issues between submitter and workers"
context: "Job attachments use S3 to transfer files between the submitter and workers. Files are uploaded during submission and downloaded to workers at session start. S3 permissions and bucket configuration are critical."
---

## Phase 1 — Triage

MUST:
- Check queue S3 bucket configuration: `aws deadline get-queue --farm-id <farm-id> --queue-id <queue-id> --query 'jobAttachmentSettings'`
- Verify S3 bucket exists and is accessible: `aws s3 ls s3://<bucket-name>/<root-prefix>/`
- Check queue IAM role S3 permissions

SHOULD:
- Verify submitter has S3 upload permissions
- Check S3 bucket policy for cross-account access if applicable
- Review job attachment size and transfer time

MAY:
- Check S3 bucket versioning and lifecycle policies
- Review S3 access logs for denied requests

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions for queue IAM role (s3:GetObject, s3:PutObject, s3:ListBucket)
- Ensure S3 bucket exists in the same region as the farm
- Fix submitter IAM permissions for S3 upload

SHOULD:
- Configure S3 bucket lifecycle rules to clean up old attachments
- Enable S3 bucket encryption for security

MAY:
- Configure S3 Transfer Acceleration for large attachments
- Set up S3 bucket metrics for monitoring transfer performance

## Safety Ratings

```
safety_ratings:
  - "get-queue: GREEN — read-only queue details"
  - "s3 ls: GREEN — read-only bucket listing"
  - "s3 bucket policy update: YELLOW — changes access permissions"
  - "s3 rm: RED — deletes objects permanently"
  - "IAM role policy update: YELLOW — changes execution permissions"
```

## Escalation Conditions

- S3 bucket access requires cross-account permissions
- Large job attachments (>100GB) causing submission timeouts
- S3 bucket policy changes require security team approval
- Attachment transfer failures affecting production render pipeline

## Data Sensitivity

- S3 bucket name and prefix: MEDIUM — reveals storage architecture
- Job attachment contents: HIGH — proprietary creative assets
- S3 access logs: MEDIUM — reveal access patterns and principals
- IAM role policies: MEDIUM — reveal permission scope

## Prohibited Actions

- NEVER make job attachment S3 bucket publicly accessible
- NEVER delete S3 objects for active or pending jobs
- NEVER modify S3 bucket policy without understanding impact on all queues
- NEVER store credentials or secrets as job attachments

## Phase 3 — Rollback

1. If S3 bucket policy was changed incorrectly, restore previous policy
2. If IAM role permissions were modified, revert to previous policy
3. If attachments were accidentally deleted, restore from S3 versioning (if enabled)
4. Resubmit failed jobs after fixing attachment access

## Common Issues

- symptoms: "Job submission fails with S3 AccessDenied during upload"
  diagnosis: "Submitter IAM role lacks s3:PutObject permission on the attachment bucket."
  resolution: "Add s3:PutObject and s3:GetObject permissions for the attachment bucket to submitter role."

- symptoms: "Worker cannot download job attachments, session fails at startup"
  diagnosis: "Queue IAM role lacks s3:GetObject permission or S3 bucket policy blocks access."
  resolution: "Fix queue IAM role S3 permissions. Verify S3 bucket policy allows the queue role."

## Output Format

```yaml
root_cause: "job_attachment — <specific_cause>"
evidence:
  - type: attachment_error
    content: "<S3 access details and error>"
severity: HIGH
mitigation:
  immediate: "Fix S3 permissions"
  long_term: "Implement attachment bucket monitoring and lifecycle policies"
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
