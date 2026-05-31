---
title: "A2 — Trust Provider Configuration"
description: "Diagnose trust provider creation and attachment issues"
status: active
severity: HIGH
triggers:
  - "trust provider error"
  - "cannot attach trust provider"
  - "trust provider configuration"
  - "IdP integration"
owner: devops-agent
objective: "Identify and resolve trust provider configuration issues"
context: "Trust providers supply identity and device context for policy evaluation. Identity providers use OIDC or IAM Identity Center. Device providers use third-party services (CrowdStrike, Jamf). Each instance supports one identity and one device provider. The policy reference name is used in Cedar policies to access context."
---

## Phase 1 — Triage

MUST:
- List trust providers: `aws ec2 describe-verified-access-trust-providers`
- Check instance trust provider attachments: `aws ec2 describe-verified-access-instances --query 'VerifiedAccessInstances[*].VerifiedAccessTrustProviders'`
- Verify trust provider configuration (OIDC issuer, authorization endpoint, token endpoint)
- Check policy reference name is set correctly

SHOULD:
- Verify OIDC provider endpoints are accessible
- Check client ID and client secret are correct
- Verify scopes include required claims (email, groups)

MAY:
- Test OIDC provider independently (token endpoint)
- Check IdP admin console for application configuration

## Phase 2 — Remediate

MUST:
- Create trust provider with correct configuration: `aws ec2 create-verified-access-trust-provider --trust-provider-type user --user-trust-provider-type oidc --oidc-options Issuer=<issuer>,AuthorizationEndpoint=<auth>,TokenEndpoint=<token>,UserInfoEndpoint=<userinfo>,ClientId=<id>,ClientSecret=<secret>,Scope="openid email profile" --policy-reference-name <name>`
- Attach to instance: `aws ec2 attach-verified-access-trust-provider --verified-access-instance-id <instance-id> --verified-access-trust-provider-id <provider-id>`

SHOULD:
- Use a descriptive policy reference name (used in Cedar policies)
- Verify OIDC provider returns expected claims

MAY:
- Configure device trust provider for enhanced security
- Test with a simple Cedar policy before complex rules

## Common Issues

- symptoms: "Trust provider attachment fails"
  diagnosis: "Instance already has a provider of the same type (identity or device)."
  resolution: "Detach existing provider first. Only one identity and one device provider per instance."

- symptoms: "OIDC authentication fails"
  diagnosis: "Incorrect OIDC endpoints, client ID, or client secret."
  resolution: "Verify all OIDC configuration matches the IdP application settings."

- symptoms: "Claims not available in policy context"
  diagnosis: "OIDC scope doesn't include required claims or wrong policy reference name."
  resolution: "Add required scopes (email, groups). Verify policy reference name in Cedar policy."

## Output Format

```yaml
root_cause: "trust_provider_config — <specific_cause>"
evidence:
  - type: provider_config
    content: "<trust provider configuration>"
  - type: attachment_status
    content: "<instance attachment status>"
  - type: oidc_test
    content: "<OIDC endpoint accessibility>"
severity: HIGH
mitigation:
  immediate: "Fix trust provider configuration"
  long_term: "Document trust provider setup and test authentication flow"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
