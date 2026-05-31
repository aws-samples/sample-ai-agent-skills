---
title: "D1 — Federated Query Failures"
description: "Diagnose and resolve Athena federated query issues"
status: active
severity: HIGH
triggers:
  - "federated query"
  - "data source"
  - "connector"
  - "Lambda"
  - "FEDERATION_ERROR"
owner: devops-agent
objective: "Identify why federated queries fail and fix connector or connectivity issues"
context: "Athena federated queries use Lambda-based connectors to query external data sources. Failures involve Lambda execution, network connectivity, IAM permissions, or connector configuration."
---

## Phase 1 — Triage

MUST:
- List data catalogs: `aws athena list-data-catalogs`
- Get data catalog details: `aws athena get-data-catalog --name <catalog-name>`
- Check Lambda function status: `aws lambda get-function --function-name <connector-function>`
- Check Lambda invocation errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Sum`

SHOULD:
- Check Lambda CloudWatch logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time $(date -u -d '1 hour ago' +%s)000 --limit 20`
- Verify Lambda VPC configuration for private data sources: `aws lambda get-function-configuration --function-name <function-name> --query 'VpcConfig'`
- Check Lambda timeout and memory settings
- Verify Lambda IAM role has access to the data source

MAY:
- Test Lambda function directly with a test event
- Check security group rules for Lambda VPC access
- Verify data source endpoint accessibility

## Phase 2 — Remediate

MUST:
- If Lambda timeout: increase timeout (up to 15 minutes) and memory
- If VPC connectivity: ensure Lambda has route to data source (security groups, NAT gateway)
- If IAM permission: add required permissions to Lambda execution role
- If connector misconfigured: update Lambda environment variables with correct connection parameters

SHOULD:
- Use VPC endpoints where available to avoid NAT gateway costs
- Set Lambda memory to at least 1024MB for connector functions
- Configure Lambda reserved concurrency for predictable performance

MAY:
- Deploy connector from AWS Serverless Application Repository for latest version
- Use Athena query federation SDK to build custom connectors

## Common Issues

- symptoms: "FEDERATION_ERROR: Lambda function timed out"
  diagnosis: "Lambda connector cannot reach the data source within the timeout period."
  resolution: "Increase Lambda timeout. Check VPC security groups and routing. Verify data source is accessible."

- symptoms: "FEDERATION_ERROR: Access denied to data source"
  diagnosis: "Lambda execution role lacks permissions to access the external data source."
  resolution: "Add data source access permissions (e.g., rds-data:ExecuteStatement, secretsmanager:GetSecretValue) to Lambda role."

## Output Format

```yaml
root_cause: "Federated query failure — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<connector error details>"
  - type: lambda_config
    content: "<VPC, timeout, memory settings>"
severity: HIGH
mitigation:
  immediate: "Fix connector configuration, permissions, or connectivity"
  long_term: "Monitor connector Lambda metrics and set up alarms for errors"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase Lambda timeout and memory | YELLOW | Affects connector performance and cost |
| Fix VPC connectivity (security groups, NAT) | YELLOW | Network configuration change |
| Add permissions to Lambda execution role | YELLOW | IAM permission change — review scope |
| Update Lambda environment variables | YELLOW | Changes connector configuration |
| Use VPC endpoints | GREEN | Network optimization — non-destructive |
| Set Lambda reserved concurrency | YELLOW | Affects connector availability |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Lambda configuration changes affect all federated queries using the connector
- VPC/security group changes may affect other resources in the same VPC

## Data Sensitivity

- Federated queries may access external databases containing sensitive data
- Lambda environment variables may contain connection strings or credentials
- VPC configuration changes may affect network isolation of sensitive data sources
- Connector logs may contain query fragments with sensitive filter values

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER store database credentials in Lambda environment variables — use Secrets Manager
- NEVER open security groups broadly to debug connectivity issues
- NEVER modify Lambda functions shared by multiple data sources without testing

## Phase 3 — Rollback

- If Lambda timeout/memory was changed: revert to previous configuration using `aws lambda update-function-configuration`
- If VPC/security group was modified: restore original network configuration
- If Lambda IAM role was changed: revert IAM policy to previous version
- If Lambda environment variables were updated: restore previous values
- Document all Lambda and network changes for audit trail

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
