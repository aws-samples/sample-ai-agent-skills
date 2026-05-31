---
title: "C1 — DynamoDB Data Source Errors"
description: "Diagnose AppSync DynamoDB data source connection and operation errors"
status: active
severity: HIGH
triggers:
  - "DynamoDB error"
  - "DynamoDB data source"
  - "GetItem failed"
  - "PutItem failed"
  - "table not found"
  - "DynamoDB access denied"
owner: devops-agent
objective: "Identify and resolve DynamoDB data source errors in AppSync resolvers"
context: "DynamoDB is the most common AppSync data source. Errors include IAM permission issues, incorrect key schema mapping, table not found, provisioned throughput exceeded, and incorrect VTL/JS mapping for DynamoDB operations (GetItem, PutItem, Query, Scan, DeleteItem, UpdateItem)."
---

## Phase 1 — Triage

MUST:
- Get the data source configuration: `aws appsync get-data-source --api-id <api-id> --name <ds-name>`
- Verify the DynamoDB table exists: `aws dynamodb describe-table --table-name <table-name>`
- Check the IAM role permissions: `aws iam get-role-policy --role-name <role-name> --policy-name <policy-name>` or `aws iam list-attached-role-policies --role-name <role-name>`
- Check the resolver mapping for correct key structure (partition key + sort key)
- Check CloudWatch Logs for specific DynamoDB errors

SHOULD:
- Verify the table region matches the AppSync API region
- Check DynamoDB throttling metrics: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ThrottledRequests --dimensions Name=TableName,Value=<table-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify the key attribute names and types match the table schema

MAY:
- Check DynamoDB Streams configuration if using event-driven patterns
- Verify GSI/LSI configuration for Query operations on indexes

## Phase 2 — Remediate

MUST:
- Ensure the IAM role has dynamodb:GetItem, PutItem, Query, Scan, DeleteItem, UpdateItem as needed
- Ensure the role trust policy allows appsync.amazonaws.com to assume it
- Fix key attribute names and types in the resolver mapping

SHOULD:
- Use `util.dynamodb.toMapValues()` (JS) or `$util.dynamodb.toMapValuesJson()` (VTL) for key serialization
- Add error handling for ConditionalCheckFailedException
- Use on-demand capacity or auto-scaling to prevent throttling

MAY:
- Enable DynamoDB point-in-time recovery for data protection
- Use batch operations for bulk data access

## Common Issues

- symptoms: "Access denied when accessing DynamoDB table"
  diagnosis: "IAM role lacks required DynamoDB permissions or doesn't trust appsync.amazonaws.com."
  resolution: "Update the IAM role with correct DynamoDB permissions and trust policy."

- symptoms: "Resolver returns null for existing items"
  diagnosis: "Key attribute names in the resolver don't match the DynamoDB table key schema."
  resolution: "Verify key names match exactly (case-sensitive). Use describe-table to confirm."

- symptoms: "ProvisionedThroughputExceededException"
  diagnosis: "DynamoDB table is throttling requests due to capacity limits."
  resolution: "Switch to on-demand capacity or increase provisioned throughput."

## Output Format

```yaml
root_cause: "dynamodb_data_source_error — <specific_cause>"
evidence:
  - type: data_source_config
    content: "<data source IAM role and table>"
  - type: table_schema
    content: "<DynamoDB table key schema>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or key mapping"
  long_term: "Add monitoring for DynamoDB throttling and error rates"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure IAM role has DynamoDB permissions | YELLOW | IAM policy change |
| Ensure role trust policy correct | YELLOW | IAM trust policy change |
| Fix key attribute names in resolver | YELLOW | Resolver code change |
| Use util.dynamodb.toMapValues() | YELLOW | Resolver code change |
| Add error handling for ConditionalCheckFailed | YELLOW | Resolver code change |
| Use on-demand capacity | YELLOW | DynamoDB capacity change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **DynamoDB table**: Table name and key schema — INTERNAL
- **IAM role**: Data source access permissions — INTERNAL
- **Query data**: User data in DynamoDB — CONFIDENTIAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER grant dynamodb:* wildcard permissions to data source roles
- NEVER modify DynamoDB table key schema (requires table recreation)

## Phase 3 — Rollback

1. Revert resolver mapping template to previous version
2. Revert IAM role policy changes
3. Revert DynamoDB capacity changes if modified
4. Verify resolver returns expected results with test queries
5. Monitor DynamoDB throttling metrics after rollback

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
