---
title: "F1 — Data Export"
description: "Diagnose data export configuration, delivery, and format issues"
status: active
severity: MEDIUM
triggers: ["data export failed", "no export data", "export configuration", "S3 export error"]
owner: devops-agent
objective: "Resolve data export issues and ensure data delivery to S3"
context: "Monitron can export sensor data to S3 for external analysis. Export is asynchronous and produces data in a Monitron-specific format. S3 bucket permissions and KMS configuration must be correct."
---

## Phase 1 — Triage

MUST:
- Verify data export is configured for the project: `aws monitron get-project --project-id <project-id>`
- Check export S3 bucket exists and is accessible: `aws s3api head-bucket --bucket <export-bucket>`
- Verify S3 bucket policy allows Monitron service to write: `aws s3api get-bucket-policy --bucket <export-bucket>`

SHOULD:
- Check if export data files exist in the bucket: `aws s3 ls s3://<export-bucket>/<prefix>/`
- Verify KMS key permissions if bucket uses SSE-KMS
- Check CloudTrail for export-related events

MAY:
- Review export data format documentation
- Check CloudWatch for export metrics

## Phase 2 — Remediate

MUST:
- Grant s3:PutObject permission to the Monitron service on the export bucket
- Ensure the S3 bucket is in the same region as the Monitron project
- Configure KMS key policy to allow Monitron service to encrypt data

SHOULD:
- Enable S3 bucket versioning for export data protection
- Set up S3 lifecycle rules for export data retention

MAY:
- Create a dedicated export bucket with pre-configured permissions
- Build custom parsers for the Monitron-specific export format

## Common Issues

- symptoms: "No data appearing in the export S3 bucket"
  diagnosis: "S3 bucket policy does not allow Monitron service to write."
  resolution: "Add s3:PutObject permission for the Monitron service principal on the export bucket."

- symptoms: "Export data exists but cannot be parsed"
  diagnosis: "Data is in Monitron-specific format, not standard CSV/JSON."
  resolution: "Use Monitron data format documentation to build custom parsers. Data is not standard CSV or JSON."

## Safety Ratings

```
safety_ratings:
  - "monitron get-project: GREEN — read-only project details"
  - "s3api head-bucket: GREEN — read-only bucket check"
  - "s3api get-bucket-policy: GREEN — read-only policy retrieval"
  - "s3api put-bucket-policy: YELLOW — modifies bucket access"
```

## Escalation Conditions

- Export bucket is in a different account requiring cross-account policy changes
- KMS key is managed by security team and requires approval for policy changes
- Export data volume exceeds S3 storage budget
- Custom parser development requires engineering resources

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals access policy
- `aws s3 ls s3://<export-bucket>/`: MEDIUM — reveals export file names
- Export data content: HIGH — contains sensor readings and equipment data
- `aws monitron get-project`: LOW — returns project metadata

## Prohibited Actions

- NEVER grant public access to the export S3 bucket
- NEVER modify KMS key policies without key administrator approval
- NEVER delete export data without confirming retention requirements
- NEVER share export data outside the organization without authorization

## Phase 3 — Rollback

1. If bucket policy was changed incorrectly, restore: `aws s3api put-bucket-policy --bucket <bucket> --policy file://previous-policy.json`
2. If KMS key policy was modified, restore from backup
3. If export was configured to wrong bucket, update project configuration
4. Verify no unauthorized access occurred during misconfiguration
5. Document the export configuration for the operations team

## Output Format

```yaml
root_cause: "data_export — <specific_cause>"
evidence:
  - type: export_error
    content: "<S3 bucket status, policy details, export configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 permissions and verify export delivery"
  long_term: "Automate export monitoring and data pipeline"
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
