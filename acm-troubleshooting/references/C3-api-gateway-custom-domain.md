---
title: "C3 — API Gateway Custom Domain Certificate"
description: "Diagnose certificate issues with API Gateway custom domain names"
status: active
severity: HIGH
triggers:
  - "API Gateway certificate"
  - "custom domain name"
  - "API Gateway SSL"
  - "API Gateway HTTPS"
  - "domain name mapping"
owner: devops-agent
objective: "Ensure ACM certificates are correctly configured for API Gateway custom domain names"
context: "API Gateway custom domain names require ACM certificates. For edge-optimized APIs, the certificate MUST be in us-east-1 (same as CloudFront). For regional APIs, the certificate must be in the same region as the API. HTTP APIs and REST APIs have different custom domain configurations."
---

## Phase 1 — Triage

MUST:
- List custom domain names: `aws apigateway get-domain-names` (REST API) or `aws apigatewayv2 get-domain-names` (HTTP API)
- Check the custom domain configuration: `aws apigateway get-domain-name --domain-name <domain>` or `aws apigatewayv2 get-domain-name --domain-name <domain>`
- Verify the certificate ARN and its status: `aws acm describe-certificate --certificate-arn <cert-arn>`
- For edge-optimized endpoints: confirm certificate is in us-east-1
- For regional endpoints: confirm certificate is in the same region as the API

SHOULD:
- Check the API mapping: `aws apigatewayv2 get-api-mappings --domain-name <domain>`
- Verify DNS points to the API Gateway domain name (CNAME or A-ALIAS record)
- Check the endpoint type (EDGE vs REGIONAL): `aws apigateway get-rest-api --rest-api-id <api-id> --query 'endpointConfiguration'`
- Verify the certificate domain matches the custom domain name

MAY:
- Check mutual TLS configuration if enabled
- Verify the base path mapping for REST APIs: `aws apigateway get-base-path-mappings --domain-name <domain>`

## Phase 2 — Remediate

MUST:
- If certificate is in wrong region: request in the correct region (us-east-1 for edge, API region for regional)
- If certificate is expired: renew or replace the certificate, then update the custom domain
- If custom domain doesn't exist: create it: `aws apigateway create-domain-name --domain-name <domain> --regional-certificate-arn <cert-arn> --endpoint-configuration types=REGIONAL`

SHOULD:
- Create DNS record pointing to the API Gateway domain name target
- Set up API mapping to connect the custom domain to the API stage
- Verify end-to-end connectivity after configuration

MAY:
- Configure mutual TLS for additional security
- Set up Route 53 health checks for the custom domain

## Common Issues

- symptoms: "Cannot create custom domain — certificate not found"
  diagnosis: "Certificate is in wrong region for the endpoint type."
  resolution: "Edge-optimized: use us-east-1 certificate. Regional: use same-region certificate."

- symptoms: "Custom domain returns 403 Forbidden"
  diagnosis: "API mapping is missing or base path is incorrect."
  resolution: "Create API mapping connecting the custom domain to the API and stage."

- symptoms: "SSL certificate error when accessing custom domain"
  diagnosis: "DNS is not pointing to the API Gateway target domain name."
  resolution: "Create CNAME or Route 53 alias record pointing to the API Gateway domain name."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Request certificate in correct region | GREEN | Creates new resource, no impact on existing |
| Create custom domain name | GREEN | New resource creation, does not affect existing APIs |
| Create API mapping | GREEN | Additive configuration |
| Create DNS record for custom domain | GREEN | Additive DNS change |

## Escalation Conditions

- Production API Gateway custom domain returning SSL errors — escalate immediately to API team
- Certificate replacement requires DNS changes affecting other services — escalate to DNS administrator
- Edge-optimized to regional endpoint migration required — escalate to architecture team
- Mutual TLS configuration changes — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws apigateway get-domain-names` | MEDIUM | Exposes custom domain configurations |
| `aws acm describe-certificate` | MEDIUM | Exposes domain names and certificate details |
| `aws apigateway get-rest-api` | MEDIUM | Exposes API configuration |

## Prohibited Actions

- **NEVER** suggest deleting certificates that are in use by AWS services
- **NEVER** suggest using self-signed certificates in production
- **NEVER** suggest disabling certificate transparency logging
- **NEVER** delete a custom domain name without confirming DNS has been updated and traffic has been redirected

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created custom domain name | Delete the custom domain: `aws apigateway delete-domain-name --domain-name <domain>` |
| Created API mapping | Delete the mapping: `aws apigatewayv2 delete-api-mapping` |
| Created DNS record | Delete the DNS record from the hosted zone |
| Requested new certificate | Delete the certificate: `aws acm delete-certificate` |

## Output Format

```yaml
root_cause: "api_gateway_certificate — <specific_cause>"
evidence:
  - type: domain_config
    content: "<get-domain-name output>"
  - type: certificate_status
    content: "<describe-certificate output>"
  - type: endpoint_type
    content: "<EDGE or REGIONAL>"
severity: HIGH
mitigation:
  immediate: "Fix certificate region or configuration"
  long_term: "Use IaC to manage custom domain and certificate together"
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
  - "NEVER suggest disabling certificate transparency"
  - "NEVER suggest using expired certificates"
  - "NEVER suggest self-signed certs for production"
