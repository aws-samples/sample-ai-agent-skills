---
title: "H1 — API Key and Service Account Issues"
description: "Diagnose API key and service account authentication problems in Amazon Managed Grafana"
status: active
severity: MEDIUM
triggers:
  - "API key"
  - "API authentication"
  - "service account"
  - "API token"
  - "401 unauthorized"
owner: devops-agent
objective: "Identify why API key or service account authentication fails and restore programmatic access"
context: "AMG supports API keys (legacy) and service accounts for programmatic access to the Grafana HTTP API. API keys are workspace-scoped and have expiration dates. Service accounts are the recommended replacement, offering token management and RBAC role assignment. Authentication failures typically stem from expired keys, incorrect tokens, or insufficient permissions."
---

## Phase 1 — Triage

MUST:
- Test API authentication: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/org`
- Check if using legacy API keys or service accounts
- List service accounts: `aws grafana list-workspace-service-accounts --workspace-id <workspace-id>`
- Verify the API key or service account token has not expired

SHOULD:
- Check the role assigned to the API key or service account (Admin, Editor, Viewer)
- Verify the workspace endpoint URL is correct
- Review CloudTrail for API access events

MAY:
- Check if the API key was created for the correct workspace
- Verify network connectivity to the workspace endpoint

## Phase 2 — Remediate

MUST:
- Create a new API key or service account token if the current one is expired
- Ensure the API key or service account has the appropriate role for the intended operations
- Use the correct workspace endpoint URL

SHOULD:
- Migrate from legacy API keys to service accounts
- Set appropriate expiration dates on tokens
- Use the minimum required role (Viewer for read-only, Editor for dashboard management)

MAY:
- Implement token rotation automation
- Set up monitoring for API authentication failures

## Common Issues

- symptoms: "API call returns 401 Unauthorized"
  diagnosis: "API key is expired, revoked, or the token value is incorrect."
  resolution: "Create a new service account token. Verify the token value is correctly included in the Authorization header."

- symptoms: "API call returns 403 Forbidden"
  diagnosis: "API key or service account has insufficient role permissions for the requested operation."
  resolution: "Create a new API key or service account with the appropriate role (e.g., Editor for dashboard creation)."

- symptoms: "API key works for one workspace but not another"
  diagnosis: "API keys are workspace-scoped. A key from one workspace cannot access another."
  resolution: "Create a separate API key or service account for each workspace that needs programmatic access."

## Safety Ratings

```
safety_ratings:
  - "list-workspace-service-accounts: GREEN — read-only enumeration"
  - "test API authentication: GREEN — read-only auth check"
  - "create new service account token: YELLOW — generates new credential"
  - "revoke/delete API key: RED — immediately removes programmatic access"
```

## Escalation Conditions

- API key compromise suspected — immediate rotation needed
- All API keys expired, blocking CI/CD pipelines
- Service account with ADMIN role needs to be created for automation
- API authentication failures affect production dashboard provisioning

## Data Sensitivity

- API keys/tokens: HIGH — provide direct programmatic access to the workspace
- Service account roles: MEDIUM — reveals automation access levels
- Workspace endpoint: LOW — workspace URL is not sensitive

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER create API keys with ADMIN role for read-only automation
- NEVER share API keys in logs, chat, or version control

## Phase 3 — Rollback

1. If an API key was created with wrong permissions, delete it and create a new one with correct role
2. If a service account was created incorrectly, delete it: `aws grafana delete-workspace-service-account --workspace-id <id> --service-account-id <sa-id>`
3. If API key rotation broke automation, restore the previous key if still valid, or update automation with the new key
4. Verify all automation and CI/CD pipelines function correctly after changes
5. Document API key rotation and update any secrets management systems

## Output Format

```yaml
root_cause: "api_key_issue — <specific_cause>"
evidence:
  - type: api_response
    content: "<HTTP status code and error message>"
  - type: service_account
    content: "<service account role and token status>"
  - type: workspace_endpoint
    content: "<workspace endpoint URL>"
severity: MEDIUM
mitigation:
  immediate: "Create new token or fix authentication configuration"
  long_term: "Migrate to service accounts with automated token rotation"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
