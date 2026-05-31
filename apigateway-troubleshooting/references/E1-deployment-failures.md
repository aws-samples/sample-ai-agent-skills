---
title: "E1 — Deployment Failures"
description: "Diagnose API Gateway deployment failures and missing changes"
status: active
severity: HIGH
triggers:
  - "deployment"
  - "deploy"
  - "changes not working"
  - "changes not visible"
  - "create-deployment"
  - "deployment failed"
owner: devops-agent
objective: "Resolve deployment failures and ensure API changes are properly deployed to stages"
context: "REST API changes require explicit deployment to take effect. Changes to methods, integrations, authorizers, models, and gateway responses are staged until deployed. HTTP API has automatic deployments by default. Deployment failures can occur due to configuration errors, circular references, or missing required fields. The most common issue is simply forgetting to deploy."
---

## Phase 1 — Triage

MUST:
- Check current deployment: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'deploymentId'`
- List recent deployments: `aws apigateway get-deployments --rest-api-id <api-id> --query 'items[*].{id:id,createdDate:createdDate,description:description}'`
- Try creating a deployment: `aws apigateway create-deployment --rest-api-id <api-id> --stage-name <stage> --description "test deployment"`
- Check for configuration errors in the API: `aws apigateway get-resources --rest-api-id <api-id>`
- Check CloudTrail for deployment events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateDeployment --max-results 10`

SHOULD:
- Verify all methods have integrations configured
- Check for circular references in models
- Verify authorizer configurations are complete
- For HTTP API, check auto-deploy setting: `aws apigatewayv2 get-stage --api-id <api-id> --stage-name '$default' --query 'AutoDeploy'`

MAY:
- Export the API to identify configuration issues: `aws apigateway get-export --rest-api-id <api-id> --stage-name <stage> --export-type swagger --accepts application/json`
- Compare current deployment with previous deployment

## Phase 2 — Remediate

MUST:
- Fix any configuration errors preventing deployment
- Create a new deployment after all changes: `aws apigateway create-deployment --rest-api-id <api-id> --stage-name <stage>`
- Verify the deployment is associated with the correct stage

SHOULD:
- Add deployment descriptions for change tracking
- Use stage variables to manage environment-specific configuration
- Implement CI/CD pipeline for automated deployments

MAY:
- Use canary deployments for gradual rollout
- Implement rollback procedures using previous deployment IDs

## Common Issues

- symptoms: "API changes not visible after modification"
  diagnosis: "REST API changes require explicit deployment. Changes are staged but not deployed."
  resolution: "Create a new deployment: aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>"

- symptoms: "Deployment fails with 'No integration defined for method'"
  diagnosis: "A method exists without an integration configured."
  resolution: "Add integration to all methods or remove methods without integrations."

- symptoms: "Deployment succeeds but old behavior persists"
  diagnosis: "Deployment created but not associated with the target stage, or caching is serving old responses."
  resolution: "Verify stage points to new deployment. Flush stage cache if caching is enabled."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
```

## Escalation Conditions

- API serves production traffic from external or internal consumers
- Fix requires redeploying a production stage
- Resolution involves modifying authorizer configuration
- Issue affects multiple APIs or is account-level throttling
- Custom domain name or certificate changes are required

## Data Sensitivity

MEDIUM - API keys and usage plan configurations control access to APIs. Authorizer configurations (Lambda authorizer code, Cognito pool settings) are security-sensitive. Access logs may contain request parameters, headers, and client IPs. Mapping templates may expose internal backend URLs and data transformation logic.

## Prohibited Actions

- NEVER suggest deleting a production stage - this immediately removes the API endpoint
- NEVER suggest removing all authorizers from an API that requires authentication
- NEVER recommend disabling API key requirements on usage-plan-protected endpoints
- NEVER suggest modifying throttling limits to unlimited on a production API
- NEVER recommend deleting and recreating a custom domain name - this causes DNS propagation downtime

## Phase 3 - Rollback

1. If a deployment was made, redeploy the previous working deployment to the stage: `aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>`
2. If authorizer configuration was changed, restore the original authorizer settings and redeploy the stage
3. If integration settings were changed, restore original integration configuration and redeploy
4. If CORS or method configuration was changed, restore original settings and redeploy the API
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "deployment — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<current deployment ID and creation date>"
  - type: configuration
    content: "<API configuration errors if any>"
  - type: cloudtrail
    content: "<deployment event history>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and create new deployment"
  long_term: "Implement CI/CD pipeline with automated deployment and validation"
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
  - "NEVER suggest removing authorizers to fix access"
  - "NEVER suggest disabling throttling"
  - "NEVER suggest open CORS for production APIs"
