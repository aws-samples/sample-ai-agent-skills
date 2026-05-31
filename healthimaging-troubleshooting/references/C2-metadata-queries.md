---
title: "C2 — Metadata Queries"
description: "Diagnose image set search and metadata query issues"
status: active
severity: MEDIUM
triggers: ["search returns no results", "metadata query failed", "search-image-sets error", "cannot find image set"]
owner: devops-agent
objective: "Resolve image set search and metadata query failures"
context: "Only indexed DICOM tags at patient, study, and series levels are searchable. Instance-level tags require per-image-set metadata retrieval."
---

## Phase 1 — Triage

MUST:
- Test search query: `aws medical-imaging search-image-sets --datastore-id <datastore-id> --search-criteria '{"filters":[{"operator":"EQUAL","values":[{"DICOMPatientId":"<patient-id>"}]}]}'`
- Verify the DICOM tag being searched is an indexed tag
- Confirm image sets exist in the datastore: `aws medical-imaging list-dicom-import-jobs --datastore-id <datastore-id>`

SHOULD:
- Check search criteria syntax matches API requirements
- Verify data was imported successfully (check import job status)

MAY:
- Retrieve metadata for a known image set to verify tag values: `aws medical-imaging get-image-set-metadata --datastore-id <datastore-id> --image-set-id <image-set-id>`

## Phase 2 — Remediate

MUST:
- Use only indexed DICOM tags in search criteria (Patient ID, Study Instance UID, Series Instance UID, etc.)
- Fix search criteria syntax
- Verify data exists in the target datastore

SHOULD:
- Implement client-side filtering for non-indexed tags
- Build a metadata index in DynamoDB for complex queries

MAY:
- Use pagination for large result sets
- Cache search results for frequently accessed studies

## Safety Ratings

```
safety_ratings:
  - "search-image-sets: GREEN — read-only search operation"
  - "get-image-set-metadata: GREEN — read-only metadata retrieval"
  - "list-dicom-import-jobs: GREEN — read-only job listing"
  - "DynamoDB index creation: YELLOW — creates new infrastructure with costs"
```

## Escalation Conditions

- Search returns no results for data confirmed to exist in the datastore
- Search performance degrades significantly with large datasets
- Required DICOM tags are not indexed, blocking clinical workflows
- Search API returns throttling errors during peak usage

## Data Sensitivity

- `aws medical-imaging search-image-sets`: MEDIUM — returns image set IDs and basic DICOM metadata
- `aws medical-imaging get-image-set-metadata`: HIGH — contains full DICOM metadata with PHI
- Search criteria: MEDIUM — may contain patient identifiers

## Prohibited Actions

- NEVER log search criteria containing patient identifiers to unprotected logs
- NEVER assume search covers all DICOM tags — verify tag is indexed
- NEVER build external metadata indexes without proper PHI encryption
- NEVER share search results containing patient data outside authorized systems

## Phase 3 — Rollback

1. If external metadata indexes were created: verify PHI encryption and access controls
2. If search criteria were modified in application code: revert to previous working queries
3. If DynamoDB tables were created for indexing: ensure encryption at rest and proper IAM policies
4. Document all metadata access for HIPAA audit trail
5. Verify search functionality is restored before closing

## Common Issues

- symptoms: "search-image-sets returns empty results for known patient"
  diagnosis: "Search tag may not be indexed, or patient ID format doesn't match imported data."
  resolution: "Verify the tag is indexed. Check exact patient ID format in imported DICOM data using get-image-set-metadata."

- symptoms: "search-image-sets returns ValidationException"
  diagnosis: "Search criteria syntax is incorrect or uses unsupported operators."
  resolution: "Review API documentation for supported search criteria format. Use EQUAL, BETWEEN operators with correct value types."

## Output Format

```yaml
root_cause: "metadata_query — <specific_cause>"
evidence:
  - type: search_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix search query"
  long_term: "Build metadata index for complex queries"
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
