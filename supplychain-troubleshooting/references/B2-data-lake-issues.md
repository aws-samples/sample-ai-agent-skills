---
title: "B2 — Data Lake Issues"
description: "Diagnose Supply Chain data lake dataset problems"
status: active
severity: MEDIUM
triggers: ["data lake error", "dataset not found", "dataset schema mismatch", "namespace error"]
owner: devops-agent
objective: "Resolve data lake dataset issues in Supply Chain"
context: "The data lake uses predefined namespaces and schemas. Datasets are organized by namespace (asc, default)."
---

## Phase 1 — Triage

MUST:
- List datasets in namespace: `aws scn list-data-lake-datasets --instance-id <instance-id> --namespace <namespace>`
- Get dataset details: `aws scn get-data-lake-dataset --instance-id <instance-id> --namespace <namespace> --name <dataset-name>`
- Verify namespace is correct (asc for managed, default for custom)

SHOULD:
- Check data integration flow status for the dataset
- Verify data format and schema compliance

MAY:
- Review dataset creation history in CloudTrail
- Check data volume and record counts

## Phase 2 — Remediate

MUST:
- Use correct namespace for dataset operations
- Fix data format to match predefined schema
- Re-run data integration flow after fixing source data

SHOULD:
- Implement schema validation in data integration flows
- Document dataset schemas for data providers

MAY:
- Create data quality monitoring dashboards
- Implement automated data validation checks

## Safety Ratings

```
safety_ratings:
  - "list-data-lake-datasets: GREEN — read-only dataset enumeration"
  - "get-data-lake-dataset: GREEN — read-only dataset details"
  - "create-data-lake-dataset: YELLOW — creates new dataset resource"
  - "delete-data-lake-dataset: RED — permanently removes dataset and data"
```

## Escalation Conditions

- Dataset schema changes required by AWS service updates
- Data quality issues affect planning accuracy
- Large datasets exceed storage quotas
- Cross-namespace data references needed but not supported

## Data Sensitivity

- `aws scn list-data-lake-datasets`: LOW — returns dataset metadata
- `aws scn get-data-lake-dataset`: LOW — returns schema and configuration
- Dataset contents: HIGH — contains proprietary supply chain data

## Prohibited Actions

- NEVER modify predefined schemas in the 'asc' namespace
- NEVER delete datasets without confirming downstream dependencies
- NEVER bypass schema validation for data ingestion
- NEVER expose dataset contents outside authorized systems

## Phase 3 — Rollback

1. If a dataset was created incorrectly: delete and recreate with correct schema
2. If data was ingested with wrong format: re-run integration flow with corrected data
3. If namespace was used incorrectly: create dataset in correct namespace
4. Verify downstream planning operations are not affected
5. Document data lake changes for audit trail

## Common Issues

- symptoms: "ListDataLakeDatasets returns empty for expected namespace"
  diagnosis: "Wrong namespace specified or datasets not yet created."
  resolution: "Verify namespace (asc vs default). Check if data integration flows have been executed."

- symptoms: "Data lake dataset has zero records after flow execution"
  diagnosis: "Source data schema doesn't match expected dataset schema."
  resolution: "Compare source data columns with dataset schema. Fix column names, types, and required fields."

## Output Format

```yaml
root_cause: "data_lake — <specific_cause>"
evidence:
  - type: dataset_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix dataset issue"
  long_term: "Implement schema validation"
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
