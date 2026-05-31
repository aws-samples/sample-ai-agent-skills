---
title: "F1 — Integration Issues"
description: "Diagnose HealthImaging integration issues with other AWS services and applications"
status: active
severity: MEDIUM
triggers: ["integration error", "S3 integration", "Lambda trigger", "API Gateway", "PACS integration"]
owner: devops-agent
objective: "Resolve integration issues between HealthImaging and other services"
context: "HealthImaging integrates with S3 for import, KMS for encryption, IAM for access, and can be accessed via SDK/API from applications."
---

## Phase 1 — Triage

MUST:
- Verify HealthImaging API endpoint connectivity: `aws medical-imaging list-datastores`
- Check IAM role trust policies for service integrations
- Verify S3 bucket configuration for import workflows: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Check VPC endpoint configuration if using private access
- Verify SDK version supports HealthImaging API

MAY:
- Test API calls with verbose logging: `aws medical-imaging list-datastores --debug`
- Check CloudWatch Logs for application-level integration errors

## Phase 2 — Remediate

MUST:
- Fix IAM role trust policies for cross-service access
- Update SDK to version supporting HealthImaging
- Fix S3 bucket policies for import job access

SHOULD:
- Implement health checks for HealthImaging API connectivity
- Use VPC endpoints for private network access

MAY:
- Set up CloudWatch dashboards for integration monitoring
- Implement circuit breaker patterns for API calls

## Safety Ratings

```
safety_ratings:
  - "list-datastores: GREEN — read-only connectivity test"
  - "s3api get-bucket-policy: GREEN — read-only policy check"
  - "VPC endpoint creation: YELLOW — creates network infrastructure"
  - "IAM role trust policy modification: YELLOW — changes service access"
```

## Escalation Conditions

- HealthImaging API is unreachable from VPC
- Integration requires cross-account access not currently configured
- SDK compatibility issues require application code changes
- PACS system integration requires custom DICOM gateway

## Data Sensitivity

- `aws medical-imaging list-datastores`: LOW — returns datastore metadata
- `aws s3api get-bucket-policy`: MEDIUM — reveals bucket access policy
- Application logs: HIGH — may contain PHI from HealthImaging operations
- VPC endpoint configuration: LOW — network metadata

## Prohibited Actions

- NEVER expose HealthImaging API endpoints publicly without authentication
- NEVER store PHI in application logs during integration debugging
- NEVER use public S3 buckets for DICOM import workflows
- NEVER bypass VPC endpoint policies for troubleshooting

## Phase 3 — Rollback

1. If VPC endpoints were created: delete if not needed: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <id>`
2. If IAM trust policies were modified: revert to previous configuration
3. If S3 bucket policies were changed: restore original policy
4. Verify all integrations are functioning after changes
5. Document integration changes for operations team

## Common Issues

- symptoms: "SDK throws UnknownServiceError for medical-imaging"
  diagnosis: "SDK version does not support HealthImaging API."
  resolution: "Update AWS SDK to latest version. HealthImaging requires recent SDK versions."

- symptoms: "Import job cannot access S3 bucket"
  diagnosis: "IAM role used by import job lacks S3 permissions or bucket policy blocks access."
  resolution: "Update IAM role with s3:GetObject permission. Check bucket policy allows the import role."

## Output Format

```yaml
root_cause: "integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix integration issue"
  long_term: "Implement integration monitoring"
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
