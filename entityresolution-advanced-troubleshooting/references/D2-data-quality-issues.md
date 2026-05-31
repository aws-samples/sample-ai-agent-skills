---
title: "D2 — Data Quality Issues"
description: "Diagnose data quality problems causing poor matching results including missing fields, inconsistent formatting, and duplicates"
status: active
severity: MEDIUM
triggers: ["poor match quality", "low match rate", "data quality", "inconsistent data", "missing fields"]
owner: devops-agent
objective: "Identify and resolve data quality issues affecting matching accuracy"
context: "Poor data quality causes poor matching results but does NOT cause job failures. Jobs complete successfully with low match rates. Data quality issues include missing fields, inconsistent formatting, duplicates, and encoding problems."
---

## Phase 1 — Triage

MUST:
- Review matching job results for match rates: `aws entityresolution get-matching-job --workflow-name <workflow-name> --job-id <job-id>`
- Check input data completeness by sampling Glue table: `aws glue get-table --database-name <db> --name <table>`
- Verify schema mapping covers the most populated fields

SHOULD:
- Sample input data to check for formatting inconsistencies
- Check for null/empty values in key matching fields
- Review data encoding (UTF-8, special characters)

MAY:
- Run data profiling on the input dataset
- Compare data quality metrics across different input sources
- Check for duplicate records in input data

## Phase 2 — Remediate

MUST:
- Standardize data formatting (addresses, names, phone numbers)
- Fill or handle missing values in key matching fields
- Fix encoding issues in input data

SHOULD:
- Implement data preprocessing pipeline before Entity Resolution
- Remove or deduplicate obvious duplicate records
- Normalize field values (uppercase/lowercase, abbreviations)

MAY:
- Use AWS Glue DataBrew for data quality profiling and transformation
- Implement data quality scoring and filtering
- Create data quality dashboards for ongoing monitoring

## Safety Ratings

```
safety_ratings:
  - "get-matching-job: GREEN — read-only job results"
  - "glue get-table: GREEN — read-only table schema"
  - "s3 get-object (data sampling): GREEN — read-only data access"
  - "Glue ETL job for data cleaning: YELLOW — modifies data"
```

## Escalation Conditions

- Data quality issues originate from upstream systems that cannot be modified
- Data volume is too small for ML matching to be effective
- Data quality improvement requires significant ETL pipeline changes
- Business requires match rates that data quality cannot support

## Data Sensitivity

- `aws entityresolution get-matching-job`: LOW — job metadata
- Input data samples: HIGH — contains PII
- Data quality metrics: LOW — aggregate statistics
- `aws glue get-table`: LOW — table schema metadata

## Prohibited Actions

- NEVER modify production input data without backup
- NEVER delete records to improve data quality without business approval
- NEVER assume data quality is acceptable without profiling
- NEVER ignore encoding issues — they cause silent matching failures

## Phase 3 — Rollback

1. If data was modified for quality improvement, restore from backup
2. If preprocessing pipeline was added, verify it does not corrupt data
3. If schema mapping was changed, revert to previous configuration
4. Re-run matching job to verify results after rollback
5. Document data quality findings and improvement plan

## Common Issues

- symptoms: "Match rate is very low despite correct configuration"
  diagnosis: "Key matching fields have many null/empty values or inconsistent formatting."
  resolution: "Profile input data. Standardize formatting. Fill missing values where possible."

- symptoms: "Many false matches on common values"
  diagnosis: "Matching fields have low cardinality (e.g., common names, cities) without additional distinguishing fields."
  resolution: "Add more fields to matching rules. Use combination of fields for matching."

- symptoms: "Records that should match are not matching"
  diagnosis: "Data formatting differences (e.g., 'St.' vs 'Street', 'Jr' vs 'Junior') prevent exact matching."
  resolution: "Normalize data before matching. Use fuzzy matching rules. Standardize abbreviations."

## Output Format

```yaml
root_cause: "data_quality — <specific_cause>"
evidence:
  - type: quality_analysis
    content: "<data quality metrics and field completeness>"
severity: MEDIUM
mitigation:
  immediate: "Fix critical data quality issues"
  long_term: "Implement data quality pipeline and monitoring"
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
