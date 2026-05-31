---
title: "D2 — SAML Federation Issues"
description: "Diagnose AppStream 2.0 SAML federation configuration problems"
status: active
severity: HIGH
triggers:
  - "SAML not working"
  - "federation error"
  - "SSO AppStream"
  - "identity provider"
owner: devops-agent
objective: "Identify and resolve SAML federation issues for AppStream"
context: "SAML federation integrates AppStream with external identity providers (Okta, Azure AD, ADFS). It requires IAM SAML provider, IAM role with trust policy, relay state URL, and correct attribute mapping. SAML and user pool cannot be used on the same stack."
---

## Phase 1 — Triage

MUST:
- Verify the IAM SAML provider exists: `aws iam list-saml-providers`
- Check the IAM role trust policy for SAML: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Verify the role has AppStream permissions
- Check the stack authentication type is SAML

SHOULD:
- Verify the SAML metadata document is current
- Check the relay state URL format: `https://appstream2.<region>.aws.amazon.com/saml?stack=<stack>&accountId=<account>`
- Verify SAML attribute mapping (NameID, SessionDuration, etc.)

MAY:
- Check IdP logs for SAML assertion errors
- Test SAML response with a SAML decoder tool

## Phase 2 — Remediate

MUST:
- Create or update the IAM SAML provider with current metadata
- Configure the IAM role trust policy to allow the SAML provider
- Set the correct relay state URL in the IdP

SHOULD:
- Map SAML attributes correctly (NameID as email, roles, session duration)
- Test with a single user before rolling out to all users

MAY:
- Configure session duration via SAML attributes
- Set up SAML debugging in the IdP

## Common Issues

- symptoms: "SAML authentication fails with 'not authorized'"
  diagnosis: "IAM role trust policy doesn't allow the SAML provider."
  resolution: "Update the role trust policy to include the SAML provider ARN."

- symptoms: "User authenticated but gets AppStream error"
  diagnosis: "Relay state URL incorrect or stack name mismatch."
  resolution: "Verify the relay state URL includes correct stack name and account ID."

- symptoms: "SAML metadata expired"
  diagnosis: "IdP certificate in the SAML metadata has expired."
  resolution: "Download new metadata from the IdP and update the IAM SAML provider."

## Output Format

```yaml
root_cause: "saml_federation_issue — <specific_cause>"
evidence:
  - type: saml_provider
    content: "<IAM SAML provider status>"
  - type: role_trust_policy
    content: "<IAM role trust policy>"
  - type: relay_state
    content: "<relay state URL configuration>"
severity: HIGH
mitigation:
  immediate: "Fix SAML configuration"
  long_term: "Document SAML setup and automate metadata rotation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create/update IAM SAML provider | YELLOW | IAM identity provider change |
| Configure IAM role trust policy | YELLOW | IAM trust policy change |
| Set correct relay state URL | YELLOW | IdP configuration change |
| Map SAML attributes | YELLOW | IdP attribute mapping change |
| Test with single user | GREEN | Non-disruptive testing |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **SAML metadata**: IdP certificate and configuration — CONFIDENTIAL
- **IAM role**: Trust policy and permissions — INTERNAL
- **Relay state URL**: Stack and account identifiers — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER modify SAML configuration during active user sessions without notice
- NEVER share SAML metadata documents with unauthorized parties

## Phase 3 — Rollback

1. Revert IAM SAML provider to previous metadata document
2. Revert IAM role trust policy to previous configuration
3. Restore previous relay state URL in the IdP
4. Revert SAML attribute mappings
5. Verify existing SAML users can still authenticate

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
