---
title: "C4 — RDS/Aurora Data Source Errors"
description: "Diagnose AppSync RDS and Aurora Serverless data source errors"
status: active
severity: HIGH
triggers:
  - "RDS data source"
  - "Aurora data source"
  - "relational database"
  - "SQL resolver"
  - "Data API"
owner: devops-agent
objective: "Identify and resolve RDS/Aurora data source errors in AppSync resolvers"
context: "AppSync connects to RDS and Aurora databases via the RDS Data API. This requires Aurora Serverless v1/v2 with the Data API enabled, or RDS Proxy. Common issues include Data API not enabled, Secrets Manager access, VPC configuration, SQL errors, and cold start latency with Aurora Serverless."
---

## Phase 1 — Triage

MUST:
- Get the data source configuration: `aws appsync get-data-source --api-id <api-id> --name <ds-name>`
- Verify the Aurora cluster exists and Data API is enabled: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].{Status:Status,HttpEndpointEnabled:HttpEndpointEnabled}'`
- Check the Secrets Manager secret is accessible: `aws secretsmanager describe-secret --secret-id <secret-arn>`
- Verify the IAM role has rds-data:ExecuteStatement and secretsmanager:GetSecretValue permissions
- Check CloudWatch Logs for SQL errors

SHOULD:
- Verify the database name and schema in the resolver configuration
- Check Aurora Serverless capacity (may be paused/scaling): `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].ScalingConfigurationInfo'`
- Test the Data API directly: `aws rds-data execute-statement --resource-arn <cluster-arn> --secret-arn <secret-arn> --database <db-name> --sql "SELECT 1"`

MAY:
- Check VPC security groups allow traffic between AppSync and the database
- Verify Aurora Serverless auto-pause settings

## Phase 2 — Remediate

MUST:
- Enable the Data API on the Aurora cluster if not enabled
- Ensure the IAM role has rds-data:* and secretsmanager:GetSecretValue permissions
- Fix SQL syntax errors in the resolver mapping

SHOULD:
- Disable auto-pause on Aurora Serverless to avoid cold start delays
- Use parameterized queries to prevent SQL injection
- Set appropriate Aurora Serverless min capacity to avoid scaling delays

MAY:
- Use connection pooling via RDS Proxy for non-Data API connections
- Add caching for frequently queried data

## Common Issues

- symptoms: "BadRequestException: HttpEndpoint is not enabled"
  diagnosis: "The Data API is not enabled on the Aurora cluster."
  resolution: "Enable the Data API: aws rds modify-db-cluster --db-cluster-identifier <id> --enable-http-endpoint"

- symptoms: "Access denied to Secrets Manager"
  diagnosis: "IAM role lacks secretsmanager:GetSecretValue permission for the database secret."
  resolution: "Add secretsmanager:GetSecretValue permission for the secret ARN."

- symptoms: "Resolver times out on first request, then works"
  diagnosis: "Aurora Serverless is paused and takes time to resume (cold start)."
  resolution: "Disable auto-pause or set minimum capacity units > 0."

## Output Format

```yaml
root_cause: "rds_aurora_data_source_error — <specific_cause>"
evidence:
  - type: data_source_config
    content: "<cluster ARN and secret ARN>"
  - type: cluster_status
    content: "<Aurora cluster status and Data API setting>"
severity: HIGH
mitigation:
  immediate: "Enable Data API, fix permissions, or fix SQL"
  long_term: "Disable auto-pause and add monitoring for database connectivity"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable Data API on Aurora cluster | YELLOW | Cluster configuration change |
| Ensure IAM role has rds-data and secrets permissions | YELLOW | IAM policy change |
| Fix SQL syntax errors | YELLOW | Resolver code change |
| Disable auto-pause on Aurora Serverless | YELLOW | Cluster configuration change |
| Use parameterized queries | YELLOW | Resolver code change |
| Set min capacity units > 0 | YELLOW | Cluster scaling change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Cluster ARN**: Database identity — INTERNAL
- **Secret ARN**: Database credentials — CONFIDENTIAL
- **SQL queries**: Database operations — INTERNAL
- **Query results**: Database content — CONFIDENTIAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER use string concatenation for SQL queries (SQL injection risk)
- NEVER expose database credentials in resolver templates

## Phase 3 — Rollback

1. Revert resolver SQL queries to previous version
2. Revert IAM role policy changes
3. Revert Aurora cluster configuration (Data API, auto-pause, capacity)
4. Verify resolver returns expected results
5. Monitor Aurora cluster metrics after rollback

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
