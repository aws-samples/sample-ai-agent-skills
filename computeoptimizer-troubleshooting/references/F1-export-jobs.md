---
title: "F1 — Export Job Failures"
description: "Diagnose recommendation export job issues"
status: active
severity: LOW
triggers: ["export failed", "export job error", "S3 export"]
owner: devops-agent
objective: "Resolve export job failures"
context: "Export jobs write recommendations to S3."
---

## Phase 1 — Triage

MUST:
- Check export jobs: `aws compute-optimizer describe-recommendation-export-jobs`
- Verify S3 bucket exists and is accessible
- Check IAM permissions for S3 write

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions
- Correct export configuration
- Retry export

## Common Issues

- symptoms: "Export job fails with AccessDenied"
  diagnosis: "IAM role cannot write to S3 bucket."
  resolution: "Update S3 bucket policy. Grant compute-optimizer write access."

## Output Format

```yaml
root_cause: "export_job — <specific_cause>"
evidence:
  - type: export_status
    content: "<export job details>"
severity: LOW
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Exports contain recommendation data, not application data |
| Blast Radius | LOW — Export failures do not affect infrastructure |
| Rollback Complexity | LOW — Retry export after fixing configuration |
| Compliance Scope | LOW — Export data is advisory |

## Escalation Conditions

- Escalate to S3 bucket owner if export fails due to bucket policy issues
- Escalate to operations if exports are needed for capacity planning reports

## Data Sensitivity

- **Export data** contains instance ARNs and utilization metrics — low sensitivity
- **S3 bucket** receiving exports should have appropriate access controls

## PROHIBITED Actions

- Do NOT export recommendation data to publicly accessible S3 buckets
- Do NOT grant Compute Optimizer broader S3 permissions than the specific export bucket

## Phase 3 — Rollback

- Delete incorrect export data from S3
- Fix S3 bucket permissions or export configuration
- Retry the export job

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
