---
title: "E3 — Canary Deployment Issues"
description: "Diagnose canary deployment configuration and traffic shifting problems"
status: active
severity: MEDIUM
triggers:
  - "canary"
  - "canary deployment"
  - "traffic shifting"
  - "canary release"
  - "gradual rollout"
owner: devops-agent
objective: "Resolve canary deployment issues including traffic splitting, promotion, and rollback"
context: "API Gateway canary deployments allow routing a percentage of traffic to a new deployment while keeping the rest on the current deployment. Canary settings include traffic percentage, stage variables overrides, and use stage cache. Canary must be explicitly promoted or deleted. Common issues include incorrect traffic percentage, canary-specific stage variables, and cache behavior."
---

## Phase 1 — Triage

MUST:
- Check canary settings: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'canarySettings'`
- Verify canary deployment ID: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query '{mainDeployment:deploymentId,canaryDeployment:canarySettings.deploymentId}'`
- Check canary traffic percentage: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'canarySettings.percentTraffic'`
- Check canary-specific stage variables: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'canarySettings.stageVariableOverrides'`
- Monitor canary vs main deployment metrics in CloudWatch

SHOULD:
- Compare error rates between canary and main deployment
- Check if canary uses stage cache: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'canarySettings.useStageCache'`
- Verify canary deployment contains expected changes

MAY:
- Check CloudTrail for canary configuration changes
- Review canary deployment history

## Phase 2 — Remediate

MUST:
- To promote canary (make it the main deployment): `aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> --patch-operations op=remove,path=/canarySettings`
- To rollback (delete canary): `aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> --patch-operations op=remove,path=/canarySettings`
- To adjust traffic: `aws apigateway update-stage --rest-api-id <api-id> --stage-name <stage> --patch-operations op=replace,path=/canarySettings/percentTraffic,value=<percent>`

SHOULD:
- Start with low canary traffic (5-10%) and gradually increase
- Monitor error rates and latency before promoting
- Use canary stage variable overrides for environment-specific testing

MAY:
- Implement automated canary analysis with CloudWatch alarms
- Use CodeDeploy for automated canary deployment management

## Common Issues

- symptoms: "Canary deployment not receiving any traffic"
  diagnosis: "Canary traffic percentage is set to 0 or canary settings are not configured."
  resolution: "Set canary traffic percentage: aws apigateway update-stage --patch-operations op=replace,path=/canarySettings/percentTraffic,value=10"

- symptoms: "All traffic going to canary instead of percentage"
  diagnosis: "Canary was promoted (canarySettings removed) making canary the main deployment."
  resolution: "This is expected after promotion. To split traffic again, create new canary settings."

- symptoms: "Canary returns different results than expected"
  diagnosis: "Canary stage variable overrides are not set or point to wrong resources."
  resolution: "Check and update canary stageVariableOverrides for the correct environment."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
2. Document all changes made during troubleshooting for audit trail
3. Verify service health after any rollback using monitoring dashboards
## Output Format

```yaml
root_cause: "canary — <specific_cause>"
evidence:
  - type: canary_settings
    content: "<canary traffic percentage, deployment ID, variables>"
  - type: metrics_comparison
    content: "<canary vs main deployment error rates>"
  - type: deployment_diff
    content: "<differences between canary and main deployment>"
severity: MEDIUM
mitigation:
  immediate: "Adjust canary traffic, promote, or rollback"
  long_term: "Implement automated canary analysis and deployment pipeline"
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
