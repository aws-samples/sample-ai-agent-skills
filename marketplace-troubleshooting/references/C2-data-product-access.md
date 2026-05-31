---
title: "C2 — Data Product Access"
description: "Diagnose data product access and delivery issues"
status: active
severity: MEDIUM
triggers: ["data product access", "data exchange error", "cannot access data", "data subscription"]
owner: devops-agent
objective: "Resolve data product access issues via AWS Data Exchange"
context: "Data products are fulfilled through AWS Data Exchange. After subscribing, buyers access data via Data Exchange APIs, S3 exports, or Lake Formation tables."
---

## Phase 1 — Triage

MUST:
- Verify Marketplace subscription: `aws marketplace-entitlement get-entitlements --product-code <product-code>`
- List Data Exchange subscriptions: `aws dataexchange list-data-sets --query 'DataSets[*].{Id:Id,Name:Name,OriginDetails:OriginDetails}'`
- Check Data Exchange permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names dataexchange:GetDataSet dataexchange:ListRevisionAssets`

SHOULD:
- List available revisions: `aws dataexchange list-data-set-revisions --data-set-id <dataset-id>`
- Check export job status: `aws dataexchange list-jobs --query 'Jobs[?Type==`EXPORT_ASSETS_TO_S3`]'`

MAY:
- Verify S3 bucket permissions for data export destination
- Check Lake Formation permissions for table-based data products

## Phase 2 — Remediate

MUST:
- Ensure both Marketplace subscription AND Data Exchange permissions are configured
- Grant dataexchange:Get*, dataexchange:List* permissions to the user
- For S3 exports, ensure destination bucket allows Data Exchange writes

SHOULD:
- Create export jobs for the latest revision: `aws dataexchange create-job --type EXPORT_ASSETS_TO_S3 --details file://export-details.json`
- Configure automatic revision notifications

MAY:
- Set up Lake Formation permissions for table-based products
- Automate data ingestion pipelines

## Safety Ratings

```
safety_ratings:
  - "marketplace-entitlement get-entitlements: GREEN — read-only check"
  - "dataexchange list-data-sets: GREEN — read-only listing"
  - "dataexchange create-job: YELLOW — initiates data export with potential storage costs"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
```

## Escalation Conditions

- Data set revisions are empty or corrupted
- Export jobs fail repeatedly with internal errors
- Data product requires Lake Formation setup not available
- Seller has not published expected data revisions

## Data Sensitivity

- `aws dataexchange list-data-sets`: LOW — returns dataset metadata
- Exported data: HIGH — may contain sensitive business or personal data
- S3 export destination: MEDIUM — reveals storage configuration
- Lake Formation tables: HIGH — may contain regulated data

## Prohibited Actions

- NEVER export data to publicly accessible S3 buckets
- NEVER share data product contents outside the terms of the subscription agreement
- NEVER disable S3 encryption on data export destinations
- NEVER grant public access to Lake Formation tables containing marketplace data

## Phase 3 — Rollback

1. Cancel pending export jobs: `aws dataexchange cancel-job --job-id <job-id>` (only if in WAITING state)
2. Delete exported data from S3 if exported to wrong location
3. Revoke Data Exchange permissions if granted too broadly
4. Remove Lake Formation permissions if configured incorrectly
5. Document data access issues for compliance records

## Common Issues

- symptoms: "Subscribed to data product but cannot see data sets"
  diagnosis: "Data Exchange permissions not configured or subscription not fully activated."
  resolution: "Grant dataexchange:GetDataSet and dataexchange:ListDataSets permissions. Wait for subscription activation."

## Output Format

```yaml
root_cause: "data_product_access — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix data access permissions"
  long_term: "Automate data ingestion pipeline"
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
