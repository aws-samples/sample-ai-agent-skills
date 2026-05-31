---
title: "B3 — Bulk Loading"
description: "Diagnose Neptune bulk loader failures and performance issues"
status: active
severity: HIGH
triggers:
  - "bulk load failed"
  - "loader error"
  - "data loading slow"
  - "bulk import"
owner: devops-agent
objective: "Identify and resolve Neptune bulk loader issues"
context: "Neptune bulk loader reads data from S3. It requires an IAM role with S3 access attached to the cluster. Supports CSV (Gremlin) and RDF formats (SPARQL). Much faster than individual query inserts."
---

## Phase 1 — Triage

MUST:
- Check loader status: `curl https://<endpoint>:8182/loader`
- Check specific load job: `curl https://<endpoint>:8182/loader/<load-id>`
- Verify IAM role attached to cluster: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].AssociatedRoles'`
- Verify S3 bucket access: `aws s3 ls s3://<bucket>/<prefix>/`

SHOULD:
- Check loader errors detail: `curl 'https://<endpoint>:8182/loader/<load-id>?details=true&errors=true'`
- Verify S3 bucket is in same region as Neptune cluster
- Check IAM role has s3:GetObject and s3:ListBucket permissions: `aws iam get-role-policy --role-name <role> --policy-name <policy>`
- Verify data format matches expected format (CSV for Gremlin, N-Triples/Turtle for SPARQL)

MAY:
- Check VPC endpoint for S3 if using private connectivity
- Monitor cluster CPU/memory during load
- Test with small data sample first

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions or S3 access issues
- Correct data format errors identified in loader error details
- Ensure S3 bucket and Neptune cluster are in the same region

SHOULD:
- Use parallelDegree parameter for faster loading
- Set failOnError to FALSE for partial loads with error tolerance
- Use updateSingleCardinalityProperties for upsert behavior

MAY:
- Split large files for parallel loading
- Implement data validation before loading

## Common Issues

- symptoms: "Loader returns LOAD_FAILED with access denied"
  diagnosis: "IAM role does not have S3 access or is not attached to the cluster."
  resolution: "Attach IAM role with s3:GetObject to the Neptune cluster."

- symptoms: "Loader fails with format error"
  diagnosis: "Data file format does not match the specified format parameter."
  resolution: "Verify CSV headers for Gremlin or RDF syntax for SPARQL data."

- symptoms: "Loading is extremely slow"
  diagnosis: "Single-threaded loading or undersized instance."
  resolution: "Increase parallelDegree parameter. Use larger instance class during load."

## Output Format

```yaml
root_cause: "bulk_loading — <specific_cause>"
evidence:
  - type: loader_status
    content: "<loader status output>"
  - type: iam_role
    content: "<IAM role configuration>"
  - type: s3_access
    content: "<S3 bucket accessibility>"
severity: HIGH
mitigation:
  immediate: "Fix loader configuration and retry"
  long_term: "Automate data loading pipeline with validation"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — bulk loading involves large datasets that may contain sensitive data
- Service Disruption Risk: HIGH — bulk loading can impact cluster performance significantly
- Compliance Impact: MEDIUM — data loading must comply with data governance policies

## Escalation Conditions
- Bulk load job fails repeatedly with data format errors
- Bulk loading causes cluster performance degradation affecting other workloads
- Load job stalls without progress for extended periods
- Data validation errors indicate data quality issues in source files
- S3 source bucket access issues prevent load completion

## Data Sensitivity
- Source data files — CONFIDENTIAL (may contain sensitive graph data)
- S3 bucket paths and IAM roles — INTERNAL
- Load job status and error logs — INTERNAL
- Data format specifications — INTERNAL
- Never expose source data samples in troubleshooting logs

## Prohibited Actions
- Do NOT run bulk loads on production clusters during peak hours without approval
- Do NOT skip data validation before initiating bulk loads
- Do NOT grant Neptune bulk load role access to unrelated S3 buckets
- Do NOT ignore bulk load error logs
- Do NOT load data from untrusted or unvalidated sources
- Do NOT bypass data governance review for new data sources

## Phase 3 — Rollback
- Cancel in-progress bulk load jobs if they are causing issues
- Restore cluster from pre-load snapshot if data integrity is compromised
- Remove partially loaded data if the load job failed midway
- Revert IAM role permissions added for the bulk load
- Verify data integrity after load completion or rollback
- Document load job outcomes and data quality issues found

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
