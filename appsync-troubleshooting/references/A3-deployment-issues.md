---
title: "A3 — AppSync Deployment Issues"
description: "Diagnose AppSync API deployment and update failures"
status: active
severity: HIGH
triggers:
  - "deployment failed"
  - "update failed"
  - "CloudFormation AppSync error"
  - "CDK deploy AppSync"
  - "SAM deploy AppSync"
owner: devops-agent
objective: "Identify and resolve issues preventing AppSync API deployment or updates"
context: "AppSync deployments can fail through direct API calls, CloudFormation, CDK, SAM, or Amplify CLI. Common issues include schema conflicts, resolver configuration errors, data source permission problems, and IaC template errors."
---

## Phase 1 — Triage

MUST:
- Check the API status: `aws appsync get-graphql-api --api-id <api-id>`
- Check schema creation status: `aws appsync get-schema-creation-status --api-id <api-id>`
- If using CloudFormation, check stack events: `aws cloudformation describe-stack-events --stack-name <stack-name> --max-items 20`
- Check CloudTrail for failed API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appsync.amazonaws.com --max-results 10`
- Verify all referenced IAM roles exist and have correct trust policies

SHOULD:
- Check if data source configurations reference valid resources (DynamoDB tables, Lambda functions)
- Verify resolver configurations match the schema (type and field names)
- Check for circular dependencies in CloudFormation/CDK templates

MAY:
- Compare current deployed configuration with the desired state
- Check if the deployment is blocked by a concurrent update

## Phase 2 — Remediate

MUST:
- Fix the specific error reported in CloudFormation events or CloudTrail
- Ensure all referenced resources (tables, functions, roles) exist before deployment
- Verify schema is valid before deploying resolvers

SHOULD:
- Deploy schema before resolvers (schema must exist for resolver attachment)
- Use CloudFormation DependsOn or CDK dependencies to order resource creation
- Test deployments in a staging environment first

MAY:
- Use AppSync API versioning for safe rollbacks
- Implement blue-green deployment patterns

## Common Issues

- symptoms: "CloudFormation CREATE_FAILED for AWS::AppSync::Resolver"
  diagnosis: "Resolver references a type or field not in the schema, or schema hasn't been created yet."
  resolution: "Add DependsOn for the schema resource, or fix the type/field name."

- symptoms: "Data source role cannot be assumed"
  diagnosis: "IAM role trust policy doesn't include appsync.amazonaws.com."
  resolution: "Update the role trust policy to allow appsync.amazonaws.com to assume it."

- symptoms: "Concurrent modification exception"
  diagnosis: "Another deployment or update is in progress for the same API."
  resolution: "Wait for the current operation to complete before retrying."

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: stack_events
    content: "<CloudFormation error events>"
  - type: api_status
    content: "<API configuration state>"
severity: HIGH
mitigation:
  immediate: "Fix the deployment error and retry"
  long_term: "Add pre-deployment validation and dependency ordering"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix specific CloudFormation error | YELLOW | IaC template change |
| Ensure referenced resources exist | GREEN | Verification |
| Verify schema is valid before deploying | GREEN | Pre-deployment check |
| Deploy schema before resolvers | GREEN | Deployment ordering |
| Test deployments in staging | GREEN | Non-production testing |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Stack events**: CloudFormation deployment details — INTERNAL
- **API configuration**: Schema, resolvers, data sources — INTERNAL
- **IAM roles**: Data source access roles — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER force-delete CloudFormation stacks with active AppSync APIs
- NEVER deploy schema changes without testing resolver compatibility

## Phase 3 — Rollback

1. Rollback CloudFormation stack: `aws cloudformation rollback-stack --stack-name <stack-name>`
2. Or update stack with previous template version
3. Verify API is functional after rollback
4. Check all resolvers are attached and working
5. Verify data source connections are healthy

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
