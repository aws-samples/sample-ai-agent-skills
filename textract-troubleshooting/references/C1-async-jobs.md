---
title: "C1 — Async Job Failures"
description: "Diagnose async document analysis job issues"
status: active
severity: HIGH
triggers:
  - "async job failed"
  - "StartDocumentAnalysis error"
  - "job not completing"
  - "GetDocumentAnalysis error"
owner: devops-agent
objective: "Identify and resolve async job failures"
context: "Async APIs process multi-page documents from S3. Jobs return JobId for result retrieval. SNS notification for completion. Results expire after 7 days."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws textract get-document-analysis --job-id <job-id> --query 'JobStatus'`
- Verify document is in S3: `aws s3 head-object --bucket <bucket> --key <key>`
- Check S3 bucket is in same region as API
- Review CloudTrail for Start* API errors

SHOULD:
- Verify IAM role for SNS notification
- Check document page count (max 3000)
- Verify document format is supported

MAY:
- Check SNS topic for notification delivery
- Monitor job processing time

## Phase 2 — Remediate

MUST:
- Upload document to S3 in same region
- Use supported format (PDF, TIFF for multi-page)
- Store results before 7-day expiration

SHOULD:
- Configure SNS notification for job completion
- Implement job status polling with backoff

MAY:
- Set up job monitoring pipeline

## Common Issues

- symptoms: "Job status FAILED"
  diagnosis: "Unsupported format, S3 access denied, or document corrupted."
  resolution: "Check document format. Verify S3 permissions. Test with smaller document."

- symptoms: "InvalidJobIdException when getting results"
  diagnosis: "Results expired (>7 days) or invalid job ID."
  resolution: "Resubmit job. Store results immediately after completion."

## Output Format

```yaml
root_cause: "async_job — <specific_cause>"
evidence:
  - type: job_status
    content: "<job status and error>"
  - type: document_info
    content: "<document location and format>"
severity: HIGH
mitigation:
  immediate: "Fix document or permissions"
  long_term: "Implement job monitoring and result persistence"
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

- Documents may contain PII, financial data, or medical records
- Extracted text preserves all sensitive content from source documents
- Identity document analysis processes government-issued IDs
- Classification: HIGHLY CONFIDENTIAL - document content may contain regulated PII

## Prohibited Actions

- NEVER process documents containing PII without confirming data handling compliance
- NEVER delete async job results before downstream consumers have retrieved them
- NEVER modify confidence thresholds in production without validation testing
- NEVER store extracted document data without appropriate encryption

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
