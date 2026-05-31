---
title: "B4 — Resolver Timeout"
description: "Diagnose resolver execution timeout issues"
status: active
severity: HIGH
triggers:
  - "resolver timeout"
  - "execution timeout"
  - "30 second"
  - "request timed out"
  - "slow resolver"
owner: devops-agent
objective: "Identify why a resolver is timing out and optimize or restructure the operation"
context: "AppSync resolvers have a hard maximum timeout of 30 seconds. This applies to individual resolvers and each function in a pipeline. Timeouts commonly occur with slow data sources, complex DynamoDB scans, unoptimized Lambda functions, or slow HTTP endpoints. The timeout cannot be increased."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch Latency metric: `aws cloudwatch get-metric-statistics --namespace AWS/AppSync --metric-name Latency --dimensions Name=GraphQLAPIId,Value=<api-id> --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Get the resolver configuration to identify the data source: `aws appsync get-resolver --api-id <api-id> --type-name <type> --field-name <field>`
- Check the data source type and configuration: `aws appsync get-data-source --api-id <api-id> --name <ds-name>`
- Enable field-level logging to see resolver execution time
- Check CloudWatch Logs for timeout errors

SHOULD:
- For DynamoDB: check if the operation is a Scan (slow) vs Query (fast)
- For Lambda: check the Lambda function's duration metric and cold start times
- For HTTP: check the external endpoint's response time
- For pipeline resolvers: identify which function in the chain is slow

MAY:
- Check if caching could reduce repeated slow calls
- Profile the data source operation independently

## Phase 2 — Remediate

MUST:
- Optimize the data source operation (use Query instead of Scan, add indexes)
- For Lambda: optimize cold starts, reduce function execution time
- For HTTP: add timeout handling, consider caching responses
- Do NOT attempt to increase the timeout beyond 30 seconds (hard limit)

SHOULD:
- Enable AppSync caching for frequently accessed, slow-to-compute data
- Use DynamoDB Query with key conditions instead of Scan with filters
- For Lambda: use provisioned concurrency to eliminate cold starts

MAY:
- Restructure as an async operation (mutation triggers async processing, subscription delivers result)
- Use batch resolvers to reduce the number of individual data source calls

## Common Issues

- symptoms: "Resolver execution timed out after 30000ms"
  diagnosis: "Data source operation exceeds 30 seconds. Common with DynamoDB Scans on large tables."
  resolution: "Optimize the query. Use Query with indexes instead of Scan. Add caching."

- symptoms: "Lambda resolver intermittently times out"
  diagnosis: "Lambda cold starts push execution time over 30 seconds."
  resolution: "Use provisioned concurrency or optimize Lambda package size."

- symptoms: "HTTP resolver timeout"
  diagnosis: "External HTTP endpoint is slow or unresponsive."
  resolution: "Add caching, implement circuit breaker pattern, or use a faster endpoint."

## Output Format

```yaml
root_cause: "resolver_timeout — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<CloudWatch Latency p99>"
  - type: resolver_config
    content: "<data source and resolver details>"
severity: HIGH
mitigation:
  immediate: "Optimize the slow data source operation"
  long_term: "Add caching, async patterns, or batch resolvers"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Optimize data source operation | YELLOW | Query/code change |
| Optimize Lambda cold starts | YELLOW | Lambda configuration change |
| Add timeout handling for HTTP | YELLOW | Resolver code change |
| Enable AppSync caching | YELLOW | Cache infrastructure change |
| Use DynamoDB Query instead of Scan | YELLOW | Resolver code change |
| Use provisioned concurrency | YELLOW | Lambda scaling change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Latency metrics**: CloudWatch p99 latency — INTERNAL
- **Resolver configuration**: Data source and resolver details — INTERNAL
- **Query patterns**: Slow query identification — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER attempt to increase the 30-second resolver timeout (hard limit)
- NEVER add full table scans to resolve timeout issues

## Phase 3 — Rollback

1. Revert resolver code to previous version
2. Revert DynamoDB index changes if GSIs were added
3. Revert Lambda provisioned concurrency settings
4. Disable caching if it was newly enabled and caused issues
5. Monitor latency metrics to confirm rollback resolved the issue

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
