---
title: "A4 — VPC Link Issues"
description: "Diagnose VPC link connectivity and configuration problems for private integrations"
status: active
severity: HIGH
triggers:
  - "VPC link"
  - "private integration"
  - "NLB"
  - "VPC endpoint"
  - "connection refused"
  - "network error"
owner: devops-agent
objective: "Resolve VPC link issues for private API Gateway integrations including NLB/ALB connectivity"
context: "VPC links enable API Gateway to access resources in a VPC. REST API uses VPC link v1 (requires NLB). HTTP API uses VPC link v2 (supports ALB, NLB, Cloud Map). VPC links must be in AVAILABLE status. Common issues include FAILED/PENDING status, security group rules, NLB target health, and subnet configuration."
---

## Phase 1 — Triage

MUST:
- Check VPC link status: `aws apigateway get-vpc-link --vpc-link-id <vpc-link-id>` (REST API v1)
- For HTTP API: `aws apigatewayv2 get-vpc-link --vpc-link-id <vpc-link-id>`
- Check integration configuration: `aws apigateway get-integration --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query '{type:type,uri:uri,connectionType:connectionType,connectionId:connectionId}'`
- Check NLB target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check NLB status: `aws elbv2 describe-load-balancers --load-balancer-arns <nlb-arn>`

SHOULD:
- Check security groups on NLB targets allow traffic from NLB subnets
- Verify NLB subnets have connectivity to target instances
- Check VPC link subnets and security groups (HTTP API v2): `aws apigatewayv2 get-vpc-link --vpc-link-id <id> --query '{Subnets:SubnetIds,SecurityGroups:SecurityGroupIds}'`

MAY:
- Check VPC flow logs for dropped traffic
- Verify DNS resolution for private endpoints
- Check NLB cross-zone load balancing setting

## Phase 2 — Remediate

MUST:
- Ensure VPC link is in AVAILABLE status (recreate if FAILED)
- Verify NLB targets are healthy and listening on the correct port
- Ensure security groups allow inbound traffic on the integration port
- For REST API: integration URI must use NLB DNS name with the VPC link connection ID

SHOULD:
- Enable cross-zone load balancing on NLB for better availability
- Use private DNS for cleaner integration URIs
- Monitor VPC link with CloudWatch metrics

MAY:
- Implement health checks on NLB target groups
- Use multiple subnets across AZs for VPC link high availability

## Common Issues

- symptoms: "500 Internal Server Error with VPC link integration"
  diagnosis: "VPC link is in FAILED or PENDING status."
  resolution: "Delete and recreate the VPC link. Check NLB exists and is active."

- symptoms: "Connection refused through VPC link"
  diagnosis: "NLB targets are unhealthy or not listening on the expected port."
  resolution: "Check target group health, security groups, and application port configuration."

- symptoms: "Intermittent 503 errors with VPC link"
  diagnosis: "Some NLB targets are unhealthy or cross-zone load balancing is disabled."
  resolution: "Fix unhealthy targets and enable cross-zone load balancing."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Network configuration changes: YELLOW - May affect connectivity"
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

1. If integration settings were changed, restore original integration configuration and redeploy
2. If CORS or method configuration was changed, restore original settings and redeploy the API
## Output Format

```yaml
root_cause: "vpc_link — <specific_cause>"
evidence:
  - type: vpc_link_status
    content: "<VPC link status and configuration>"
  - type: nlb_health
    content: "<NLB target health status>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix VPC link status, NLB targets, or security groups"
  long_term: "Implement health monitoring and multi-AZ redundancy"
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
