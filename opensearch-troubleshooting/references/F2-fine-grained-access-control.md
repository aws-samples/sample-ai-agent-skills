---
title: "F2 — OpenSearch Fine-Grained Access Control (FGAC)"
description: "Diagnose and resolve fine-grained access control permission issues"
status: active
severity: HIGH
triggers:
  - "FGAC"
  - "fine-grained access"
  - "security_exception"
  - "role mapping"
  - "internal user"
  - "backend role"
  - "Cognito"
  - "SAML"
owner: devops-agent
objective: "Resolve FGAC permission issues and configure proper role mappings"
context: "Fine-grained access control (FGAC) provides cluster-internal authorization with index-level, document-level, and field-level permissions. FGAC is evaluated AFTER the domain access policy allows the request. Authentication can use the internal user database, SAML, or Amazon Cognito. IAM roles must be mapped to FGAC backend roles. The master user has full access. Common issue: IAM role allowed by access policy but not mapped to any FGAC role."
---

## Phase 1 — Triage

MUST:
- Check FGAC status: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.AdvancedSecurityOptions'`
- Check role mappings: `curl -XGET "https://<endpoint>/_plugins/_security/api/rolesmapping?pretty" -u <master-user>:<password>`
- Check available roles: `curl -XGET "https://<endpoint>/_plugins/_security/api/roles?pretty" -u <master-user>:<password>`
- Check current user info: `curl -XGET "https://<endpoint>/_plugins/_security/authinfo?pretty"`
- Test with master user: `curl -XGET "https://<endpoint>/_cluster/health?pretty" -u <master-user>:<password>`

SHOULD:
- Check internal users: `curl -XGET "https://<endpoint>/_plugins/_security/api/internalusers?pretty" -u <master-user>:<password>`
- Check Cognito configuration if used: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.CognitoOptions'`
- Check action groups: `curl -XGET "https://<endpoint>/_plugins/_security/api/actiongroups?pretty" -u <master-user>:<password>`

MAY:
- Check audit logs if enabled for denied requests
- Review SAML configuration if using SAML authentication

## Phase 2 — Remediate

MUST:
- Map IAM role to FGAC backend role: `curl -XPUT "https://<endpoint>/_plugins/_security/api/rolesmapping/all_access" -H 'Content-Type: application/json' -u <master-user>:<password> -d '{"backend_roles":["arn:aws:iam::<account>:role/<role-name>"]}'`
- If locked out: use master user credentials to fix role mappings
- If Cognito misconfigured: verify Cognito user pool and identity pool settings

SHOULD:
- Create custom roles with least-privilege permissions: `curl -XPUT "https://<endpoint>/_plugins/_security/api/roles/my-read-role" -H 'Content-Type: application/json' -u <master-user>:<password> -d '{"cluster_permissions":["cluster_composite_ops_ro"],"index_permissions":[{"index_patterns":["my-index-*"],"allowed_actions":["read"]}]}'`
- Map IAM roles to custom roles instead of all_access
- Document role mappings and access patterns

MAY:
- Implement document-level security for multi-tenant indices
- Use field-level security to restrict sensitive fields
- Set up audit logging to track access patterns

## Common Issues

- symptoms: "security_exception: no permissions for [indices:data/read/search]"
  diagnosis: "IAM role not mapped to any FGAC role with search permissions."
  resolution: "Map the IAM role ARN to an appropriate FGAC backend role."

- symptoms: "Access works with master user but not with IAM role"
  diagnosis: "IAM role allowed by access policy but not mapped in FGAC."
  resolution: "Add IAM role ARN to the appropriate FGAC role mapping."

- symptoms: "Cognito login works but no index access"
  diagnosis: "Cognito group not mapped to FGAC role with index permissions."
  resolution: "Map Cognito group to FGAC role with appropriate index permissions."

## Output Format

```yaml
root_cause: "fgac — <specific_cause>"
evidence:
  - type: fgac_config
    content: "<FGAC enabled, auth method>"
  - type: role_mappings
    content: "<current role mappings>"
  - type: error
    content: "<security_exception details>"
severity: HIGH
mitigation:
  immediate: "Map IAM role or user to appropriate FGAC role"
  long_term: "Implement least-privilege FGAC roles, document access patterns"
```


## Safety Ratings
```
safety_ratings:
  - "Check FGAC status and role mappings: GREEN — read-only API calls"
  - "Check internal users: GREEN — read-only inspection"
  - "Map IAM role to FGAC role: YELLOW — grants cluster-internal permissions"
  - "Create custom FGAC roles: YELLOW — defines new permission boundaries"
  - "Map to all_access role: RED — grants full cluster access"
```

## Escalation Conditions
- Domain serves production search
- Users locked out of cluster (need master user to fix)
- FGAC role mapping changes affecting multiple teams
- Cognito or SAML integration issues
- Document-level or field-level security changes

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "FGAC role mappings: IAM role to permission mappings"
    - "Internal user credentials: usernames and password hashes"
    - "Master user credentials: full cluster access"
    - "Cognito/SAML configuration: authentication details"
  handling: "NEVER expose master user credentials. FGAC role mappings reveal access patterns."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER expose or log master user credentials
- NEVER map untrusted IAM roles to all_access
- NEVER disable FGAC once enabled (cannot be disabled)

## Phase 3 — Rollback
- If role mapping was changed: restore previous mapping using master user
- If custom role was created: delete the role if not needed
- If internal user was created: delete the user
- If Cognito group mapping was changed: restore previous mapping

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
