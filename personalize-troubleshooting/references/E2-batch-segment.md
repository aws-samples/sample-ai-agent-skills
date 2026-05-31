---
title: "E2 — Batch Segment Job Issues"
description: "Diagnose batch segment job failures"
status: active
severity: MEDIUM
triggers: ["batch segment failed", "CreateBatchSegmentJob error", "user segmentation error"]
owner: devops-agent
objective: "Identify and resolve batch segment job failures"
context: "Batch segment jobs identify users likely to interact with specific items. Requires USER_SEGMENTATION recipe. Input is item IDs, output is user segments."
---

## Phase 1 — Triage

MUST:
- Check job: `aws personalize describe-batch-segment-job --batch-segment-job-arn <arn>`
- Verify solution uses USER_SEGMENTATION recipe
- Check S3 input format (JSON lines with itemId)
- Verify S3 permissions

SHOULD:
- Verify solution version is ACTIVE
- Check input item IDs exist in dataset
- Review job failure reason

MAY:
- Test with single item first
- Check numResults parameter

## Phase 2 — Remediate

MUST:
- Use solution with USER_SEGMENTATION recipe
- Format input as JSON lines: {"itemId":"<id>"}
- Grant S3 access to IAM role

SHOULD:
- Validate item IDs exist in dataset
- Set appropriate numResults

MAY:
- Implement segmentation pipeline

## Common Issues

- symptoms: "Batch segment fails with recipe error"
  diagnosis: "Solution doesn't use USER_SEGMENTATION recipe."
  resolution: "Create solution with Item-Affinity or Item-Action-Affinity recipe."

- symptoms: "Empty segments returned"
  diagnosis: "Item IDs not in dataset or insufficient interaction data."
  resolution: "Verify item IDs. Add more interaction data."

## Output Format

```yaml
root_cause: "batch_segment — <specific_cause>"
evidence:
  - type: job_status
    content: "<batch segment job details>"
severity: MEDIUM
mitigation:
  immediate: "Fix recipe or input format"
  long_term: "Implement segmentation pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — batch segmentation involves user profiling
- Service Disruption Risk: LOW — batch failures do not affect real-time operations
- Compliance Impact: HIGH — user segmentation must comply with privacy regulations

## Escalation Conditions
- Batch segment job fails with data errors
- Segmentation produces biased or discriminatory results
- Segment output contains PII
- Batch job exceeds expected cost
- Segment results do not match expected criteria

## Data Sensitivity
- User segment data — RESTRICTED
- Segmentation criteria — CONFIDENTIAL
- Batch output — CONFIDENTIAL
- User profiling data — RESTRICTED
- Never expose individual user segment assignments

## Prohibited Actions
- Do NOT create segments based on protected characteristics
- Do NOT store segment output in publicly accessible locations
- Do NOT share segment data with unauthorized parties
- Do NOT ignore segmentation bias
- Do NOT process user data without consent
- Do NOT retain segment data beyond approved retention periods

## Phase 3 — Rollback
- Delete incorrect segment output
- Re-run segmentation with corrected parameters
- Restore previous segmentation configurations
- Verify segment quality after re-run
- Document segmentation issues and compliance findings
- Update segmentation procedures based on findings

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
