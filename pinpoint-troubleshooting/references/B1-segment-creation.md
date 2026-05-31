---
title: "B1 — Segment Creation Issues"
description: "Diagnose segment creation and import failures"
status: active
severity: MEDIUM
triggers:
  - "segment creation failed"
  - "import segment error"
  - "segment empty"
  - "cannot create segment"
owner: devops-agent
objective: "Identify and resolve segment creation failures"
context: "Segments can be created dynamically (attribute-based) or imported from S3. Common failures: invalid filter criteria, S3 access issues, format errors in import files, or endpoint quota exceeded."
---

## Phase 1 — Triage

MUST:
- List segments: `aws pinpoint get-segments --application-id <app-id>`
- Check specific segment: `aws pinpoint get-segment --application-id <app-id> --segment-id <seg-id>`
- For import segments, check import job: `aws pinpoint get-import-jobs --application-id <app-id>`
- Verify endpoint count: `aws pinpoint get-application-settings --application-id <app-id>`

SHOULD:
- Check S3 bucket access for import segments
- Verify import file format (CSV or JSON)
- Check for endpoint quota limits

MAY:
- Review CloudTrail for CreateSegment events
- Check for duplicate endpoints

## Phase 2 — Remediate

MUST:
- Fix import file format (CSV headers or JSON schema)
- Grant S3 read access to Pinpoint IAM role
- Remove duplicate endpoints to stay within quota

SHOULD:
- Validate import files before uploading
- Use dynamic segments for attribute-based targeting

MAY:
- Implement segment size monitoring

## Common Issues

- symptoms: "Import job completes but segment is empty"
  diagnosis: "Import file format incorrect or all records failed validation."
  resolution: "Check import job failures count. Fix CSV headers or JSON format."

- symptoms: "Dynamic segment shows 0 endpoints"
  diagnosis: "Filter criteria too restrictive or endpoints not yet propagated."
  resolution: "Broaden filter criteria. Wait for endpoint propagation."

## Output Format

```yaml
root_cause: "segment_creation — <specific_cause>"
evidence:
  - type: segment_status
    content: "<segment details>"
  - type: import_job
    content: "<import job status>"
severity: MEDIUM
mitigation:
  immediate: "Fix segment configuration or import file"
  long_term: "Implement import file validation pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — segments contain customer demographic and behavioral data
- Service Disruption Risk: LOW — segment issues do not affect message delivery
- Compliance Impact: HIGH — segmentation must comply with data protection regulations

## Escalation Conditions
- Segment creation fails with validation errors
- Segment includes customers who should be excluded
- Segment size unexpectedly large or small
- Segment criteria produce incorrect results
- Imported segments contain invalid data

## Data Sensitivity
- Customer attributes — RESTRICTED
- Segment criteria — CONFIDENTIAL
- Customer counts — INTERNAL
- Behavioral data — RESTRICTED
- Never expose individual customer data from segments

## Prohibited Actions
- Do NOT create segments based on protected characteristics for discriminatory purposes
- Do NOT include customers who have opted out
- Do NOT share segment data with unauthorized parties
- Do NOT create segments without data governance review
- Do NOT ignore segment validation warnings
- Do NOT export segment data to unsecured locations

## Phase 3 — Rollback
- Delete incorrectly created segments
- Restore previous segment configurations
- Revert segment criteria changes
- Verify segment membership after changes
- Document segment issues and data governance findings
- Update segmentation procedures

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
