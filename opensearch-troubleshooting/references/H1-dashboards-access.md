---
title: "H1 — OpenSearch Dashboards Access Issues"
description: "Diagnose and resolve OpenSearch Dashboards login and access problems"
status: active
severity: MEDIUM
triggers:
  - "Dashboards access"
  - "Dashboards login"
  - "Kibana access"
  - "cannot access Dashboards"
  - "Dashboards 403"
  - "Cognito login"
owner: devops-agent
objective: "Restore access to OpenSearch Dashboards"
context: "OpenSearch Dashboards is the visualization and management UI for OpenSearch. Access is controlled by the domain access policy, FGAC (if enabled), and optionally Amazon Cognito authentication. For VPC domains, Dashboards is only accessible from within the VPC. The Dashboards endpoint is at https://<domain-endpoint>/_dashboards. Common issues include access policy denials, FGAC role mapping, Cognito misconfiguration, and VPC connectivity."
---

## Phase 1 — Triage

MUST:
- Check Dashboards endpoint: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.{Endpoint:Endpoint,DashboardEndpoint:DashboardEndpoint,Endpoints:Endpoints}'`
- Check access policy: `aws opensearch describe-domain-config --domain-name <domain> --query 'DomainConfig.AccessPolicies.Options'`
- Check FGAC status: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.AdvancedSecurityOptions'`
- Check Cognito configuration: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.CognitoOptions'`
- Check if VPC domain: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.VPCOptions'`

SHOULD:
- Test Dashboards URL: `curl -XGET "https://<endpoint>/_dashboards" -v 2>&1 | grep -E "HTTP|302|403|401"`
- Check Cognito user pool if using Cognito: `aws cognito-idp describe-user-pool --user-pool-id <pool-id>`
- Check Cognito identity pool: `aws cognito-identity describe-identity-pool --identity-pool-id <pool-id>`

MAY:
- Check browser developer tools for specific error messages
- Check Cognito user attributes and group membership
- Review CloudTrail for authentication events

## Phase 2 — Remediate

MUST:
- If VPC domain: access from within VPC (bastion, VPN, or reverse proxy)
- If access policy blocking: update policy to allow Dashboards access
- If FGAC blocking: map user/role to appropriate FGAC role (see F2)
- If Cognito misconfigured: verify user pool, identity pool, and domain association

SHOULD:
- Set up Nginx reverse proxy for external Dashboards access to VPC domains
- Configure Cognito user groups mapped to FGAC roles for multi-user access
- Use SAML for enterprise SSO integration

MAY:
- Set up AWS Client VPN for developer access to VPC Dashboards
- Configure custom Dashboards branding and default index patterns
- Enable Dashboards audit logging

## Common Issues

- symptoms: "Dashboards shows blank page or 403"
  diagnosis: "Access policy or FGAC denying access to Dashboards resources."
  resolution: "Update access policy. Map user to FGAC role with kibana_user or equivalent."

- symptoms: "Cognito login redirects but fails"
  diagnosis: "Cognito identity pool not configured or IAM role mapping incorrect."
  resolution: "Verify Cognito identity pool authenticated role has es:ESHttp* permissions."

- symptoms: "Cannot reach Dashboards URL"
  diagnosis: "VPC domain — Dashboards not accessible from public internet."
  resolution: "Access from within VPC. Set up reverse proxy or VPN for external access."

## Output Format

```yaml
root_cause: "dashboards_access — <specific_cause>"
evidence:
  - type: dashboards_endpoint
    content: "<Dashboards URL and access test>"
  - type: auth_config
    content: "<FGAC, Cognito, access policy>"
  - type: connectivity
    content: "<VPC/public, network access>"
severity: MEDIUM
mitigation:
  immediate: "Fix access policy, FGAC mapping, or network connectivity"
  long_term: "Document Dashboards access architecture, set up persistent access path"
```


## Safety Ratings
```
safety_ratings:
  - "Check Dashboards endpoint and configuration: GREEN — read-only API calls"
  - "Test Dashboards URL: GREEN — read-only connectivity test"
  - "Update access policy for Dashboards: YELLOW — changes domain access"
  - "Map user to FGAC role: YELLOW — grants Dashboards permissions"
  - "Set up reverse proxy: YELLOW — creates new access path"
```

## Escalation Conditions
- Domain serves production search
- Dashboards access blocked for operations team
- Cognito authentication issues requiring identity pool changes
- VPC domain requiring proxy or VPN setup for Dashboards access
- FGAC role mapping changes affecting multiple users

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Dashboards endpoint: access URL"
    - "Cognito configuration: user pool and identity pool details"
    - "FGAC role mappings: user access patterns"
  handling: "Dashboards provides visual access to all index data. Restrict access appropriately."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER expose Dashboards to the public internet without authentication
- NEVER map all users to all_access FGAC role

## Phase 3 — Rollback
- If access policy was updated: restore previous policy
- If FGAC role mapping was changed: revert using master user
- If Cognito configuration was modified: restore previous settings
- If reverse proxy was set up: remove proxy configuration

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
