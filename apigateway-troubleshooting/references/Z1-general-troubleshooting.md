---
title: "Z1 — General API Gateway Troubleshooting (Catch-All)"
description: "Fallback SOP for API Gateway issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown API Gateway issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step. Covers REST API, HTTP API, and WebSocket API types."
---

## Phase 1 — Triage

MUST:
- Identify API type (REST, HTTP, WebSocket): `aws apigateway get-rest-apis` and `aws apigatewayv2 get-apis`
- Get API overview: `aws apigateway get-rest-api --rest-api-id <api-id>` or `aws apigatewayv2 get-api --api-id <api-id>`
- Check stages: `aws apigateway get-stages --rest-api-id <api-id>` or `aws apigatewayv2 get-stages --api-id <api-id>`
- Check error rates: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 4XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check 5xx errors: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name 5XXError --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check latency: `aws cloudwatch get-metric-statistics --namespace AWS/ApiGateway --metric-name Latency --dimensions Name=ApiName,Value=<api-name> --start-time <start> --end-time <end> --period 300 --statistics p99`
- Test basic connectivity: `curl -sI https://<api-url>/`

SHOULD:
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=apigateway.amazonaws.com --max-results 10`
- Check resources and methods: `aws apigateway get-resources --rest-api-id <api-id>`
- Check execution logs if enabled: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --start-time <epoch-ms> --limit 20`
- Check current deployment: `aws apigateway get-stage --rest-api-id <api-id> --stage-name <stage> --query 'deploymentId'`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Lambda proxy/custom integration errors → Integration (A1-A4)
- 4xx/5xx/CORS/binary errors → Errors (B1-B4)
- Authorization/authentication failures → Authorization (C1-C3)
- 429 Too Many Requests → Throttling (D1-D3)
- Changes not visible / deployment issues → Deployment (E1-E3)
- Custom domain / certificate problems → Custom Domains (F1-F2)
- WebSocket connection / routing / delivery → WebSocket (G1-G3)
- Missing logs / log configuration → Logging (H1-H2)
- Mapping template / validation errors → Request/Response (I1-I2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all integrations for the affected resources
- Review authorizer configurations
- Check gateway responses for custom error handling
- Verify deployment is current and contains expected changes
- Check throttling at all levels (account, stage, method, usage plan)

SHOULD:
- Enable execution logging if not already enabled
- Enable X-Ray tracing for detailed request flow analysis
- Check AWS Health Dashboard for API Gateway service events
- Compare with a known-good API configuration

MAY:
- Export API definition: `aws apigateway get-export --rest-api-id <api-id> --stage-name <stage> --export-type swagger --accepts application/json`
- Check WAF if associated with the stage
- Review resource policy for IP/VPC restrictions

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
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
3. If throttling settings were modified, restore original rate/burst limits: `aws apigateway update-stage --rest-api-id <id> --stage-name <stage> --patch-operations <revert>`
4. If integration settings were changed, restore original integration configuration and redeploy
5. If custom domain or base path mapping was changed, restore original mapping configuration
6. If CORS or method configuration was changed, restore original settings and redeploy the API
7. If WebSocket route configuration was changed, restore original route integration and redeploy
8. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<integration|errors|authorization|throttling|deployment|custom_domains|websocket|logging|request_response|unknown>"
investigation_path: "API overview → CloudWatch → execution logs → <domain_classification>"
evidence:
  - type: api_config
    content: "<API configuration summary>"
  - type: cloudwatch
    content: "<error rates and latency metrics>"
  - type: execution_logs
    content: "<relevant log entries>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
