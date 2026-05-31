---
title: "D2 — Data Source Connectors"
description: "Diagnose and resolve Athena data source connector issues"
status: active
severity: MEDIUM
triggers:
  - "connector"
  - "data source registration"
  - "catalog not found"
  - "connector deployment"
owner: devops-agent
objective: "Identify data source connector deployment and configuration issues"
context: "Athena connectors are Lambda functions deployed from SAR or custom-built. They translate Athena queries into source-specific API calls."
---

## Phase 1 — Triage

MUST:
- List registered data catalogs: `aws athena list-data-catalogs --query 'DataCatalogsSummary[*].{Name:CatalogName,Type:Type}'`
- Get catalog configuration: `aws athena get-data-catalog --name <catalog-name>`
- Check connector Lambda function: `aws lambda get-function-configuration --function-name <function-name>`
- Verify Lambda environment variables: `aws lambda get-function-configuration --function-name <function-name> --query 'Environment.Variables'`

SHOULD:
- Check connector version and update availability
- Verify spill bucket configuration for large result sets
- Check Lambda execution role permissions

MAY:
- Test connector with a simple query (SELECT 1 or SHOW TABLES)
- Check SAR for latest connector version

## Phase 2 — Remediate

MUST:
- If catalog not found: register data catalog with `aws athena create-data-catalog`
- If Lambda missing: deploy connector from SAR or redeploy
- If environment variables wrong: update Lambda configuration with correct connection parameters
- If spill bucket missing: create and configure spill S3 bucket

SHOULD:
- Use Secrets Manager for database credentials instead of environment variables
- Configure appropriate Lambda memory and timeout for the data source
- Set up CloudWatch alarms on connector Lambda errors

MAY:
- Deploy multiple connector instances for different data sources
- Use connector-specific tuning parameters for performance

## Common Issues

- symptoms: "Catalog not found: <catalog-name>"
  diagnosis: "Data catalog not registered in Athena."
  resolution: "Register with: aws athena create-data-catalog --name <name> --type LAMBDA --parameters function=arn:aws:lambda:..."

- symptoms: "Connector returns empty results for known data"
  diagnosis: "Connection parameters (host, port, database) incorrect in Lambda environment."
  resolution: "Update Lambda environment variables with correct connection details."

## Output Format

```yaml
root_cause: "Data source connector — <specific_issue>"
evidence:
  - type: catalog_config
    content: "<catalog registration details>"
  - type: lambda_config
    content: "<connector function configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix connector registration or configuration"
  long_term: "Use IaC for connector deployment with validated configurations"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Register data catalog | GREEN | New resource creation — non-destructive |
| Deploy connector from SAR | YELLOW | Deploys Lambda function — review configuration |
| Update Lambda environment variables | YELLOW | Changes connector configuration |
| Create spill S3 bucket | GREEN | New resource creation — non-destructive |
| Set up CloudWatch alarms | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Connector deployment affects all queries using the data catalog
- Environment variable changes may contain connection credentials

## Data Sensitivity

- Data catalog registration reveals external data source locations
- Connector configuration may contain database connection details
- Spill bucket may temporarily contain query result data from external sources
- Lambda environment variables may contain sensitive connection parameters

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER store credentials in Lambda environment variables — use Secrets Manager
- NEVER deploy connectors without reviewing the SAR application permissions

## Phase 3 — Rollback

- If data catalog was registered incorrectly: delete with `aws athena delete-data-catalog`
- If Lambda connector was deployed incorrectly: delete the Lambda function and associated resources
- If environment variables were changed: restore previous values
- Document all catalog and connector changes for audit trail

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
  - "NEVER suggest disabling query result encryption"
  - "NEVER suggest overly broad S3 access for workgroups"
  - "NEVER suggest disabling workgroup enforcement"
