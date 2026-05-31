---
title: "D2 — User Trust Evaluation"
description: "Diagnose user trust evaluation issues with OIDC and IAM Identity Center"
status: active
severity: HIGH
triggers: ["user trust failed", "OIDC token error", "identity center not working", "user context missing", "authentication failed"]
owner: devops-agent
objective: "Resolve user trust evaluation issues in Verified Access"
context: "User trust providers (OIDC, IAM Identity Center) supply user identity context to Cedar policies. Issues can stem from token configuration, claim mapping, or provider availability."
---

## Phase 1 — Triage

MUST:
- Check user trust provider configuration: `aws ec2 describe-verified-access-trust-providers --query 'VerifiedAccessTrustProviders[?TrustProviderType==\`user\`].{Id:VerifiedAccessTrustProviderId,UserType:UserTrustProviderType,PolicyRef:PolicyReferenceName}'`
- Review access logs for user context: `aws logs filter-log-events --log-group-name <va-log-group> --filter-pattern '{ $.activity = "denied" }' --start-time <epoch-ms>`
- Verify OIDC provider endpoint is reachable (for OIDC type)

SHOULD:
- Check OIDC discovery document at issuer URL/.well-known/openid-configuration
- Verify client ID and scopes are correct
- Confirm IAM Identity Center application is properly configured (for SSO type)

MAY:
- Decode OIDC tokens to verify claim contents
- Test authentication flow in a browser developer tools

## Phase 2 — Remediate

MUST:
- Ensure OIDC issuer URL is correct and reachable
- Verify client ID matches the OIDC application configuration
- Confirm requested scopes include claims needed by Cedar policies

SHOULD:
- Test authentication with a known-good user account
- Verify token expiration and refresh behavior

MAY:
- Configure additional OIDC scopes for richer user context
- Implement token caching monitoring

## Common Issues

- symptoms: "User gets authentication error before reaching Cedar policy evaluation"
  diagnosis: "OIDC provider configuration mismatch (wrong issuer URL, client ID, or scopes)."
  resolution: "Verify OIDC configuration: aws ec2 describe-verified-access-trust-providers. Check issuer URL, client ID, and scopes match the OIDC provider application."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-trust-providers: GREEN — read-only enumeration"
  - "Access log analysis: GREEN — read-only log review"
  - "ModifyVerifiedAccessTrustProvider: YELLOW — changes authentication configuration"
  - "OIDC token inspection: YELLOW — involves authentication credentials"
```

## Escalation Conditions

- OIDC provider is down or returning invalid tokens
- IAM Identity Center configuration requires organizational changes
- Token claims do not include required attributes for Cedar policies
- Authentication failures affect all users (provider-wide issue)

## Data Sensitivity

- User trust provider configuration: MEDIUM — reveals authentication setup
- OIDC tokens: HIGH — contain user identity claims (email, groups, name)
- Access logs with user context: HIGH — contain personal identity information
- OIDC client secrets: HIGH — authentication credentials

## Prohibited Actions

- NEVER expose OIDC client secrets or user tokens in logs or documentation
- NEVER modify OIDC provider configuration without a rollback plan
- NEVER share decoded token contents containing PII
- NEVER disable user trust evaluation to bypass authentication issues

## Phase 3 — Rollback

1. Revert trust provider configuration to previous settings
2. If OIDC application was modified at the provider, restore previous configuration
3. Verify authentication works for test users after rollback
4. Check access logs to confirm user context is populated correctly
5. Document the authentication issue and resolution

## Output Format

```yaml
root_cause: "user_trust — <specific_cause>"
evidence:
  - type: user_context
    content: "<trust provider config and authentication details>"
severity: HIGH
mitigation:
  immediate: "Fix user trust configuration"
  long_term: "Implement authentication monitoring and alerting"
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
