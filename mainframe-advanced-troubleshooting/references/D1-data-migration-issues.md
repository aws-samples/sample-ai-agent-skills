---
title: "D1 — Data Migration Issues"
description: "Diagnose data migration issues from mainframe to AWS M2"
status: active
severity: CRITICAL
triggers: ["data migration failed", "EBCDIC conversion error", "VSAM migration", "data corruption", "copybook parsing error"]
owner: devops-agent
objective: "Resolve data migration issues including EBCDIC conversion, copybook parsing, and dataset transfer"
context: "Data migration from mainframes requires EBCDIC-to-ASCII conversion, copybook-based record parsing, and dataset type handling (VSAM, sequential, GDG). Incorrect migration causes data corruption."
---

## Phase 1 — Triage

MUST:
- Check application data configuration: `aws m2 get-application --application-id <app-id>`
- Review data-related logs: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "data" --start-time <epoch-ms>`
- Verify S3 data files exist: `aws s3 ls s3://<data-bucket>/<data-prefix>/`

SHOULD:
- Verify copybook definitions match the actual data record layouts
- Check EBCDIC code page used for conversion
- Validate sample records after conversion

MAY:
- Compare record counts between source and target
- Check for packed decimal and binary field conversion accuracy

## Phase 2 — Remediate

MUST:
- Verify copybooks accurately describe the data record layout
- Use the correct EBCDIC code page for the source mainframe
- Validate data integrity after conversion (record counts, checksums, sample verification)

SHOULD:
- Test migration with a representative data sample before full migration
- Handle packed decimal (COMP-3) and binary (COMP) fields explicitly

MAY:
- Implement automated data validation scripts
- Create data reconciliation reports

## Common Issues

- symptoms: "Numeric fields contain garbage data after migration"
  diagnosis: "Packed decimal (COMP-3) or binary (COMP) fields not converted correctly."
  resolution: "Verify copybook field definitions. Ensure packed decimal and binary fields are handled by the conversion tool. Re-run conversion with correct field type handling."

## Safety Ratings

```
safety_ratings:
  - "get-application: GREEN — read-only application check"
  - "s3 ls: GREEN — read-only S3 listing"
  - "Data conversion: YELLOW — transforms data, potential for corruption"
  - "Data upload to S3: YELLOW — overwrites existing data files"
```

## Escalation Conditions

- Data corruption detected after migration affecting business operations
- Copybook definitions do not match actual data layouts
- EBCDIC code page unknown or non-standard
- Large-scale data migration (TB+) with integrity concerns

## Data Sensitivity

- Mainframe datasets: HIGH — contain business-critical data
- Copybook definitions: MEDIUM — reveal data structures
- Converted data files: HIGH — contain business data in new format
- Migration logs: MEDIUM — may reference data content

## Prohibited Actions

- NEVER overwrite source data during migration without verified backups
- NEVER assume EBCDIC conversion is correct without validation
- NEVER skip copybook verification before data migration
- NEVER migrate production data without testing with a representative sample first

## Phase 3 — Rollback

1. If migrated data is corrupted, delete the corrupted files from S3
2. Re-extract data from the source mainframe with corrected conversion settings
3. Validate the re-extracted data before uploading
4. Update application configuration to point to corrected data
5. Document the data migration issue and corrected conversion parameters

## Output Format

```yaml
root_cause: "data_migration — <specific_cause>"
evidence:
  - type: migration_error
    content: "<conversion error details and data validation results>"
severity: CRITICAL
mitigation:
  immediate: "Fix data conversion issue"
  long_term: "Implement automated data validation pipeline"
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
