---
title: "D1 — API Key Issues"
description: "Diagnose AppSync API key authorization failures"
status: active
severity: HIGH
triggers:
  - "API key"
  - "x-api-key"
  - "UnauthorizedException"
  - "API key expired"
  - "missing API key"
owner: devops-agent
objective: "Identify and resolve API key authorization issues in AppSync"
context: "API_KEY is the simplest AppSync auth mode. Keys are passed via the x-api-key header. Common issues include expired keys (max 365 days), missing header, wrong key value, and multi-auth configuration where API_KEY isn't the default mode and types lack @aws_api_key directives."
---

## Phase 1 — Triage

MUST:
- List API keys and check expiration: `aws appsync list-api-keys --api-id <api-id>`
- Check the API auth configuration: `aws appsync get-graphql-api --api-id <api-id> --query 'graphqlApi.{defaultAuth:authenticationType,additionalAuth:additionalAuthenticationProviders}'`
- Verify the client is sending the x-api-key header with the correct value
- Check if the API key has expired (compare `expires` timestamp with current time)

SHOULD:
- If multi-auth, check if the target type has @aws_api_key directive
- Check CloudWatch Logs for specific authorization error messages
- Verify the API key belongs to the correct API

MAY:
- Check if there are multiple API keys and the client is using the wrong one
- Verify the API key hasn't been deleted and recreated with a different value

## Phase 2 — Remediate

MUST:
- If expired, create a new API key: `aws appsync create-api-key --api-id <api-id> --expires <epoch-timestamp>`
- Ensure the client sends the correct x-api-key header
- If multi-auth, add @aws_api_key directive to types that need API key access

SHOULD:
- Set up API key rotation before expiration (max 365 days)
- Use CloudWatch alarms to alert before API key expiration
- Consider migrating to Cognito or IAM auth for production workloads

MAY:
- Automate API key rotation using Lambda and Secrets Manager
- Use multiple API keys for different clients/environments

## Common Issues

- symptoms: "UnauthorizedException: You are not authorized"
  diagnosis: "API key is expired or missing from the request."
  resolution: "Check key expiration with list-api-keys. Create a new key if expired."

- symptoms: "API key works for Query but not Mutation"
  diagnosis: "Multi-auth API where Mutation type doesn't have @aws_api_key directive."
  resolution: "Add @aws_api_key directive to the Mutation type in the schema."

- symptoms: "API key suddenly stopped working"
  diagnosis: "Key expired (default 7 days if not configured). Check the expires field."
  resolution: "Create a new API key with a longer expiration (up to 365 days)."

## Output Format

```yaml
root_cause: "api_key_issue — <specific_cause>"
evidence:
  - type: api_key_status
    content: "<key ID and expiration>"
  - type: auth_config
    content: "<API authentication configuration>"
severity: HIGH
mitigation:
  immediate: "Create new API key or fix client header"
  long_term: "Implement key rotation and expiration monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create new API key | YELLOW | New credential creation |
| Ensure client sends correct header | GREEN | Client configuration |
| Add @aws_api_key directive | YELLOW | Schema change |
| Set up API key rotation | GREEN | Security improvement |
| Consider migrating to Cognito/IAM auth | YELLOW | Auth mode change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **API keys**: Authentication credentials — CONFIDENTIAL
- **Auth configuration**: API authentication modes — INTERNAL
- **Schema directives**: Authorization directives — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER share API keys in logs, code repositories, or public channels
- NEVER use API_KEY auth as the sole auth mode for production APIs with sensitive data

## Phase 3 — Rollback

1. Delete the newly created API key if it was created in error
2. Restore previous API key if it was rotated
3. Revert schema directive changes (@aws_api_key)
4. Update clients to use the previous API key
5. Verify all clients can authenticate after rollback

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
