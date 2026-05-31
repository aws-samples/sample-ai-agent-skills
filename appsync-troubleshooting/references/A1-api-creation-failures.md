---
title: "A1 — AppSync API Creation Failures"
description: "Diagnose why an AppSync GraphQL API fails to create or configure"
status: active
severity: HIGH
triggers:
  - "API creation failed"
  - "cannot create GraphQL API"
  - "create-graphql-api error"
  - "AppSync API not created"
owner: devops-agent
objective: "Identify why an AppSync API cannot be created and resolve the configuration or permission issue"
context: "AppSync API creation can fail due to IAM permission issues, service limits, invalid configuration parameters, or CloudWatch Logs role misconfiguration. The API requires specific IAM permissions and valid authentication configuration."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for the caller: `aws sts get-caller-identity`
- Verify AppSync service limits: `aws service-quotas get-service-quota --service-code appsync --quota-code L-06CAE498`
- List existing APIs to check limit usage: `aws appsync list-graphql-apis`
- Check CloudTrail for the failed creation event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appsync.amazonaws.com --max-results 10`
- Verify the authentication configuration is valid (API key, Cognito pool ARN, OIDC issuer URL)

SHOULD:
- Check if the CloudWatch Logs role ARN is valid: `aws iam get-role --role-name <log-role-name>`
- Verify the log role has the correct trust policy for appsync.amazonaws.com
- Check if the region supports AppSync

MAY:
- Check AWS Health Dashboard for AppSync service issues
- Verify VPC configuration if using private APIs

## Phase 2 — Remediate

MUST:
- Ensure the IAM principal has `appsync:CreateGraphQLApi` permission
- Provide a valid authentication configuration (at least one auth mode)
- If using CloudWatch Logs, ensure the role ARN exists and trusts appsync.amazonaws.com

SHOULD:
- Request a service limit increase if at the API limit
- Use CloudFormation or CDK for repeatable API creation

MAY:
- Enable field-level logging after API creation for debugging

## Common Issues

- symptoms: "AccessDeniedException when calling CreateGraphQLApi"
  diagnosis: "IAM principal lacks appsync:CreateGraphQLApi permission."
  resolution: "Add appsync:CreateGraphQLApi to the IAM policy."

- symptoms: "InvalidParameterException for authentication type"
  diagnosis: "Auth mode configuration is incomplete (e.g., Cognito pool ARN missing)."
  resolution: "Provide all required parameters for the chosen auth mode."

- symptoms: "LimitExceededException"
  diagnosis: "Account has reached the maximum number of AppSync APIs per region (default 25)."
  resolution: "Delete unused APIs or request a service limit increase."

## Output Format

```yaml
root_cause: "api_creation_failure — <specific_cause>"
evidence:
  - type: cloudtrail_event
    content: "<error from creation attempt>"
  - type: iam_permissions
    content: "<caller permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the identified configuration or permission issue"
  long_term: "Use IaC templates with validated configurations"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure IAM principal has CreateGraphQLApi | YELLOW | IAM policy change |
| Provide valid authentication configuration | GREEN | Configuration fix |
| Ensure CloudWatch Logs role exists | GREEN | Verification |
| Request service limit increase | GREEN | Quota management |
| Use CloudFormation or CDK | GREEN | IaC recommendation |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **API configuration**: Authentication mode, endpoints — INTERNAL
- **IAM permissions**: Caller identity and policies — INTERNAL
- **CloudTrail events**: API creation audit trail — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER create APIs with no authentication in production
- NEVER expose GraphQL introspection in production without access control

## Phase 3 — Rollback

1. Delete the newly created API if it caused issues: `aws appsync delete-graphql-api --api-id <api-id>`
2. Revert IAM policy changes if permissions were modified
3. Remove CloudWatch Logs role if it was created for this API
4. Verify no dependent resources reference the deleted API
5. Restore previous API configuration if an update was made

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
