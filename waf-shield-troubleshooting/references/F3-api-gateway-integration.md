---
title: "F3 — API Gateway + WAF Integration Issues"
description: "Diagnose WAF integration problems with API Gateway"
status: active
severity: HIGH
triggers:
  - "API Gateway WAF"
  - "APIGW WAF"
  - "API Gateway integration"
  - "WAF not protecting API"
  - "REST API WAF"
owner: devops-agent
objective: "Identify and fix WAF integration issues with API Gateway REST APIs"
context: "WAF can be associated with API Gateway REST API stages using REGIONAL scope web ACLs. HTTP APIs and WebSocket APIs do not support direct WAF association — use CloudFront in front of them for WAF protection. The web ACL must be in the same region as the API Gateway. WAF evaluates requests after API Gateway receives them but before method execution."
---

## Phase 1 — Triage

MUST:
- Check if a web ACL is associated with the API stage: `aws wafv2 get-web-acl-for-resource --resource-arn arn:aws:apigateway:<region>::/restapis/<api-id>/stages/<stage-name>`
- Verify the web ACL scope is REGIONAL: `aws wafv2 get-web-acl --name <acl-name> --scope REGIONAL --id <acl-id>`
- Verify the API type is REST API (HTTP APIs and WebSocket APIs don't support WAF directly)
- Check CloudWatch WAF metrics for the web ACL

SHOULD:
- Verify the API stage exists and is deployed
- Check if API Gateway has its own request validation that may conflict with WAF
- Review API Gateway access logs for correlation

MAY:
- Check CloudTrail for CreateStage or UpdateStage events
- Verify the API Gateway resource policy is not conflicting with WAF

## Phase 2 — Remediate

MUST:
- Associate the web ACL with the REST API stage: `aws wafv2 associate-web-acl --web-acl-arn <web-acl-arn> --resource-arn arn:aws:apigateway:<region>::/restapis/<api-id>/stages/<stage-name>`
- Ensure the web ACL is REGIONAL scope and in the same region
- For HTTP APIs, put CloudFront in front and use a CLOUDFRONT scope web ACL

SHOULD:
- Configure WAF rules appropriate for API traffic (JSON body inspection, API key validation)
- Enable WAF logging for API request analysis
- Set up rate-based rules to protect API endpoints

MAY:
- Use API Gateway usage plans and throttling in addition to WAF rate limiting
- Implement API Gateway request validation as a complement to WAF

## Common Issues

- symptoms: "Cannot associate WAF with API Gateway"
  diagnosis: "API is an HTTP API or WebSocket API (only REST APIs support WAF). Or scope/region mismatch."
  resolution: "Use REST API for direct WAF. For HTTP APIs, use CloudFront + CLOUDFRONT scope WAF."

- symptoms: "WAF associated but API requests not being evaluated"
  diagnosis: "Web ACL is associated with the wrong stage, or the stage is not deployed."
  resolution: "Verify the stage name in the resource ARN. Redeploy the API stage."

- symptoms: "WAF blocks API requests with 403 but API Gateway returns its own 403 format"
  diagnosis: "WAF 403 response format differs from API Gateway's default 403."
  resolution: "Use WAF custom response bodies to return API-compatible error format (JSON)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Associate web ACL with API stage | YELLOW | Enables WAF; reversible |
| Configure WAF for API traffic | YELLOW | Rule modification; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- API Gateway stage configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl-for-resource` | LOW | Association status |
| `get-web-acl` | LOW | Rule configuration |
| CloudWatch WAF metrics | LOW | Aggregate request counts |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER disassociate WAF from a production API Gateway without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Web ACL association | Disassociate via `disassociate-web-acl` |
| WAF rule configuration | Revert via `update-web-acl` |

## Output Format

```yaml
root_cause: "api_gateway_integration — <specific_cause>"
evidence:
  - type: association
    content: "<web ACL association status>"
  - type: api_type
    content: "<REST, HTTP, or WebSocket>"
  - type: stage
    content: "<stage name and deployment status>"
severity: HIGH
mitigation:
  immediate: "Associate web ACL or fix API type/scope mismatch"
  long_term: "Implement comprehensive API protection with WAF and API Gateway features"
```

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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"
