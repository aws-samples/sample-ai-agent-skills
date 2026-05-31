---
title: "B2 — SSO Integration"
description: "Diagnose SSO and SAML integration issues"
status: active
severity: HIGH
triggers: ["SSO not working", "SAML error", "identity provider issue", "federation failed"]
owner: devops-agent
objective: "Resolve SSO and SAML federation issues for Wickr authentication"
context: "Wickr supports IAM Identity Center (SSO) and SAML 2.0 federation for user authentication."
---

## Phase 1 — Triage

MUST:
- Verify IAM Identity Center configuration: `aws sso-admin list-instances`
- Check SAML provider configuration: `aws iam list-saml-providers`
- Review CloudTrail for authentication failures: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20`

SHOULD:
- Verify SAML metadata document is current and certificate is not expired
- Check IAM Identity Center application assignment for affected users
- Test SSO login flow in an incognito browser window

MAY:
- Review identity provider logs for SAML assertion errors
- Check SAML attribute mapping configuration

## Phase 2 — Remediate

MUST:
- Update expired SAML certificates: `aws iam update-saml-provider --saml-provider-arn <arn> --saml-metadata-document file://metadata.xml`
- Fix SAML attribute mapping if user attributes are not passed correctly
- Ensure IAM Identity Center application is properly configured

SHOULD:
- Configure SAML assertion signing and encryption
- Set up fallback authentication method for admin access

MAY:
- Implement just-in-time user provisioning via SAML
- Configure session duration and timeout settings

## Safety Ratings

```
safety_ratings:
  - "sso-admin list-instances: GREEN — read-only SSO enumeration"
  - "iam list-saml-providers: GREEN — read-only SAML listing"
  - "iam update-saml-provider: YELLOW — modifies authentication configuration"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- All users unable to authenticate via SSO simultaneously
- SAML provider certificate expired with no backup
- IAM Identity Center instance is misconfigured affecting multiple applications
- SSO integration broken after identity provider upgrade

## Data Sensitivity

- `aws iam list-saml-providers`: LOW — returns SAML provider ARNs
- SAML metadata documents: MEDIUM — contains federation configuration
- Authentication logs: MEDIUM — reveals user login patterns
- SAML assertions: HIGH — contain user identity attributes

## Prohibited Actions

- NEVER delete the SAML provider without a replacement configured
- NEVER share SAML metadata documents externally
- NEVER disable SSO without an alternative authentication method
- NEVER modify SAML configuration during business hours without a maintenance window

## Phase 3 — Rollback

1. If SAML provider was updated, restore previous metadata document
2. If IAM Identity Center configuration was changed, restore original settings
3. Verify all users can authenticate after rollback
4. Confirm no unauthorized access occurred during the outage
5. Document the SSO issue and resolution for the security team

## Common Issues

- symptoms: "SAML authentication fails with 'Invalid SAML response'"
  diagnosis: "SAML assertion attributes do not match expected mapping."
  resolution: "Verify SAML attribute mapping matches Wickr requirements (email, name attributes)."

- symptoms: "Users can SSO to other apps but not Wickr"
  diagnosis: "Wickr application not properly configured in IAM Identity Center."
  resolution: "Verify Wickr application configuration and user/group assignments in IAM Identity Center."

## Output Format

```yaml
root_cause: "sso_integration — <specific_cause>"
evidence:
  - type: authentication_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore SSO authentication"
  long_term: "Implement certificate rotation and monitoring"
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
