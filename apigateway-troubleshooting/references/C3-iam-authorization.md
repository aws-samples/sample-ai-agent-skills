---
title: "C3 — IAM Authorization Issues"
description: "Diagnose IAM authorization failures including SigV4 signing, IAM policies, and resource policies"
status: active
severity: HIGH
triggers:
  - "IAM auth"
  - "SigV4"
  - "Signature Version 4"
  - "IAM authorization"
  - "credential"
  - "access denied"
  - "resource policy"
owner: devops-agent
objective: "Resolve IAM authorization failures by validating SigV4 signing, IAM policies, and resource policies"
context: "IAM authorization requires requests to be signed with AWS Signature Version 4. API Gateway evaluates both the caller's IAM policy (execute-api:Invoke) and the API's resource policy. Both must allow the request. Cross-account access requires both the caller's IAM policy AND the API resource policy to explicitly allow. Common failures include incorrect signing, missing IAM permissions, or resource policy denials."
---

## Phase 1 — Triage

MUST:
- Verify method uses IAM authorization: `aws apigateway get-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'authorizationType'`
- Check API resource policy: `aws apigateway get-rest-api --rest-api-id <api-id> --query 'policy'`
- Check caller's IAM policy for execute-api:Invoke permission
- Check execution logs for auth failure details: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "denied" --start-time <epoch-ms>`
- Verify the request is properly SigV4 signed (check Authorization header format)

SHOULD:
- Use IAM Policy Simulator to test permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names execute-api:Invoke --resource-arns arn:aws:execute-api:<region>:<account>:<api-id>/<stage>/<method>/<path>`
- Check for explicit deny in any applicable policy (SCPs, permission boundaries)
- Verify the signing region matches the API region

MAY:
- Check CloudTrail for API Gateway authorization events
- Verify temporary credentials (STS) are not expired
- Check if VPC endpoint policy restricts access

## Phase 2 — Remediate

MUST:
- Ensure caller IAM policy allows: `execute-api:Invoke` on `arn:aws:execute-api:<region>:<account>:<api-id>/<stage>/<method>/<path>`
- For cross-account: both caller IAM policy AND API resource policy must allow
- Ensure SigV4 signing uses the correct region, service name (execute-api), and credentials
- Check for explicit denies in resource policy that override allows

SHOULD:
- Use least-privilege IAM policies with specific API/stage/method resources
- Implement resource policy for IP-based or VPC-based restrictions
- Use IAM roles (not long-term credentials) for API access

MAY:
- Implement cross-account access with resource policy and caller role
- Use VPC endpoint policies for additional access control

## Common Issues

- symptoms: "403 Forbidden with IAM auth and valid credentials"
  diagnosis: "IAM policy allows but resource policy denies, or vice versa."
  resolution: "Both IAM policy AND resource policy must allow. Check for explicit denies in resource policy."

- symptoms: "403 with cross-account IAM auth"
  diagnosis: "Cross-account requires BOTH caller IAM policy AND API resource policy to allow."
  resolution: "Add caller account/role to API resource policy AND ensure caller IAM policy allows execute-api:Invoke."

- symptoms: "403 with 'The security token included in the request is invalid'"
  diagnosis: "SigV4 signing error: wrong region, expired credentials, or incorrect service name."
  resolution: "Verify signing region, credential validity, and service name is 'execute-api'."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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

1. If a deployment was made, redeploy the previous working deployment to the stage: `aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>`
2. If authorizer configuration was changed, restore the original authorizer settings and redeploy the stage
3. If integration settings were changed, restore original integration configuration and redeploy
4. If CORS or method configuration was changed, restore original settings and redeploy the API
5. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "iam_auth — <specific_cause>"
evidence:
  - type: iam_policy
    content: "<caller IAM policy for execute-api>"
  - type: resource_policy
    content: "<API resource policy>"
  - type: signing_details
    content: "<SigV4 signing configuration>"
severity: HIGH
mitigation:
  immediate: "Fix IAM policy, resource policy, or SigV4 signing"
  long_term: "Implement least-privilege policies and cross-account access patterns"
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
