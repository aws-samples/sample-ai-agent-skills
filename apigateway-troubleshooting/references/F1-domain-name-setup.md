---
title: "F1 — Custom Domain Name Setup"
description: "Diagnose custom domain name configuration issues including base path mappings and DNS"
status: active
severity: HIGH
triggers:
  - "custom domain"
  - "domain name"
  - "base path"
  - "base path mapping"
  - "DNS"
  - "CNAME"
  - "Route 53"
owner: devops-agent
objective: "Resolve custom domain name setup issues including domain creation, base path mappings, and DNS configuration"
context: "Custom domain names allow using your own domain (api.example.com) instead of the default execute-api URL. Edge-optimized domains use CloudFront (ACM cert in us-east-1). Regional domains use regional endpoints (ACM cert in same region). Base path mappings route requests to specific APIs and stages. DNS must point to the API Gateway domain name."
---

## Phase 1 — Triage

MUST:
- Check custom domain configuration: `aws apigateway get-domain-name --domain-name <domain>`
- Check base path mappings: `aws apigateway get-base-path-mappings --domain-name <domain>`
- Verify DNS resolution: `nslookup <domain>` or `dig <domain>`
- Check domain endpoint type (EDGE vs REGIONAL): `aws apigateway get-domain-name --domain-name <domain> --query 'endpointConfiguration'`
- Verify ACM certificate status: `aws acm describe-certificate --certificate-arn <cert-arn>`

SHOULD:
- Check if domain name status is AVAILABLE: `aws apigateway get-domain-name --domain-name <domain> --query 'domainNameStatus'`
- Verify the target domain name for DNS: `aws apigateway get-domain-name --domain-name <domain> --query '{regional:regionalDomainName,edge:distributionDomainName}'`
- Check Route 53 records if using Route 53: `aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Name=='<domain>.']"`

MAY:
- Check mutual TLS configuration if enabled
- Verify ownership certificate for domain

## Phase 2 — Remediate

MUST:
- Ensure ACM certificate is in the correct region (us-east-1 for edge, same region for regional)
- Ensure ACM certificate is ISSUED status and covers the domain name
- Create base path mapping to connect domain to API/stage: `aws apigateway create-base-path-mapping --domain-name <domain> --rest-api-id <api-id> --stage <stage> --base-path <path>`
- Configure DNS to point to the API Gateway domain name (CNAME or Route 53 alias)

SHOULD:
- Use Route 53 alias records instead of CNAME for apex domains
- Use regional endpoint type for lower latency in single-region deployments
- Configure multiple base path mappings for different APIs under one domain

MAY:
- Implement mutual TLS for additional security
- Use API mapping for HTTP API: `aws apigatewayv2 create-api-mapping --domain-name <domain> --api-id <api-id> --stage '$default'`

## Common Issues

- symptoms: "Custom domain returns 403 Forbidden"
  diagnosis: "Base path mapping is missing or points to wrong API/stage."
  resolution: "Create or fix base path mapping: aws apigateway create-base-path-mapping --domain-name <domain> --rest-api-id <id> --stage <stage>"

- symptoms: "Domain creation fails with certificate error"
  diagnosis: "ACM certificate is in wrong region or not in ISSUED status."
  resolution: "For edge: use us-east-1 cert. For regional: use same-region cert. Ensure cert is validated."

- symptoms: "DNS resolves but connection times out"
  diagnosis: "DNS points to wrong target or domain name status is not AVAILABLE."
  resolution: "Update DNS to point to the correct API Gateway domain name. Wait for domain to become AVAILABLE."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
2. If integration settings were changed, restore original integration configuration and redeploy
3. If custom domain or base path mapping was changed, restore original mapping configuration
4. If WebSocket route configuration was changed, restore original route integration and redeploy
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "custom_domain — <specific_cause>"
evidence:
  - type: domain_config
    content: "<domain name, endpoint type, status>"
  - type: base_path_mappings
    content: "<base path to API/stage mappings>"
  - type: dns_resolution
    content: "<DNS records and resolution>"
  - type: certificate
    content: "<ACM certificate status and region>"
severity: HIGH
mitigation:
  immediate: "Fix domain configuration, base path mapping, or DNS"
  long_term: "Implement infrastructure-as-code for domain management"
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
