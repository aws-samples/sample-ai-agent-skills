---
title: "D1 — Batch Translation Job Issues"
description: "Diagnose batch translation job failures"
status: active
severity: HIGH
triggers:
  - "batch job failed"
  - "StartTextTranslationJob error"
  - "batch translation not completing"
owner: devops-agent
objective: "Identify and resolve batch translation job failures"
context: "Batch translation processes documents from S3. Supports TXT, HTML, DOCX, XLIFF. Requires IAM role with S3 permissions. Jobs can take minutes to hours."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws translate describe-text-translation-job --job-id <job-id>`
- Verify S3 input/output locations exist
- Check IAM data access role permissions
- Review job error details

SHOULD:
- Verify input file format matches ContentType
- Check file count in input prefix
- Verify language pair is supported

MAY:
- Check for concurrent job limits
- Review partial results in output location

## Phase 2 — Remediate

MUST:
- Fix S3 permissions on data access role
- Use supported input format (TXT, HTML, DOCX, XLIFF)
- Specify correct ContentType

SHOULD:
- Test with single file before batch
- Monitor job progress

MAY:
- Implement batch job orchestration

## Common Issues

- symptoms: "Job fails with access denied"
  diagnosis: "IAM role lacks S3 read/write permissions."
  resolution: "Add s3:GetObject and s3:PutObject to data access role."

- symptoms: "Job completes but some files not translated"
  diagnosis: "Individual file format errors."
  resolution: "Check per-file error details in job output. Fix file formats."

## Output Format

```yaml
root_cause: "batch_job — <specific_cause>"
evidence:
  - type: job_status
    content: "<job details>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or file format"
  long_term: "Implement batch job monitoring"
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

- Source documents may contain business-sensitive content
- Custom terminologies reveal domain-specific vocabulary
- Parallel data contains paired translation examples
- Classification: CONFIDENTIAL - business content and proprietary terminology

## Prohibited Actions

- NEVER delete custom terminologies used by production translation workflows
- NEVER modify parallel data sets during active batch translation jobs
- NEVER change formality settings on production translation endpoints without review
- NEVER process documents containing PII without confirming data handling compliance

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
