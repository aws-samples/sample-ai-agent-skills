---
title: "D2 — Dataset Management"
description: "Diagnose dataset management issues in M2 environments"
status: active
severity: MEDIUM
triggers: ["dataset not found", "file access error", "VSAM error", "GDG issue", "dataset allocation"]
owner: devops-agent
objective: "Resolve dataset management issues including VSAM, sequential, GDG, and PDS handling"
context: "M2 applications access datasets that map to mainframe file types. VSAM (KSDS, ESDS, RRDS), sequential files, GDG, and PDS each have specific handling requirements."
---

## Phase 1 — Triage

MUST:
- Check application data configuration: `aws m2 get-application --application-id <app-id>`
- Review file access errors in logs: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "file" --start-time <epoch-ms>`
- Verify dataset files exist in S3: `aws s3 ls s3://<data-bucket>/<dataset-prefix>/`

SHOULD:
- Verify dataset type mapping (VSAM type, record format, key definitions)
- Check file handler configuration in the application definition
- Verify dataset permissions and access modes

MAY:
- Compare dataset configuration with source mainframe catalog entries
- Check for dataset contention between concurrent jobs

## Phase 2 — Remediate

MUST:
- Ensure dataset files exist at the configured S3 paths
- Verify dataset type definitions match the actual file format
- Fix file handler configuration for the specific dataset type

SHOULD:
- Implement dataset catalog management for tracking file locations
- Configure appropriate access modes (read, write, read-write)

MAY:
- Implement automated dataset lifecycle management
- Create dataset backup and recovery procedures

## Common Issues

- symptoms: "VSAM file open error with status code 35"
  diagnosis: "VSAM file not found at the configured path."
  resolution: "Verify the VSAM file exists in S3 at the path specified in the application definition. Check file handler configuration for correct dataset mapping."

## Safety Ratings

```
safety_ratings:
  - "get-application: GREEN — read-only application check"
  - "s3 ls: GREEN — read-only S3 listing"
  - "Dataset configuration changes: YELLOW — affects data access patterns"
  - "Dataset deletion: RED — permanently removes data files"
```

## Escalation Conditions

- Critical datasets missing or corrupted
- VSAM key structure mismatch causing data access failures
- GDG generation management issues affecting batch processing
- Dataset contention causing application deadlocks

## Data Sensitivity

- Dataset configuration: MEDIUM — reveals data architecture
- Dataset contents: HIGH — contain business data
- File handler configuration: LOW — reveals file access patterns
- S3 bucket paths: MEDIUM — reveals data storage structure

## Prohibited Actions

- NEVER delete datasets without verified backups
- NEVER modify VSAM key definitions on production datasets
- NEVER change dataset access modes without understanding concurrent access patterns
- NEVER assume all mainframe dataset types are handled identically in M2

## Phase 3 — Rollback

1. Restore deleted or corrupted datasets from S3 versioning or backups
2. Revert file handler configuration changes
3. Verify application can access all required datasets after rollback
4. Check batch job and CICS transaction dataset access
5. Document the dataset issue and resolution

## Output Format

```yaml
root_cause: "dataset_management — <specific_cause>"
evidence:
  - type: dataset_error
    content: "<file access error and dataset configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix dataset access issue"
  long_term: "Implement dataset catalog and lifecycle management"
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
