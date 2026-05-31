---
title: "D1 — Identity Trust Provider Issues"
description: "Diagnose identity trust provider (OIDC/IAM Identity Center) problems"
status: active
severity: HIGH
triggers:
  - "identity provider error"
  - "OIDC error"
  - "IAM Identity Center"
  - "authentication failed"
owner: devops-agent
objective: "Identify and resolve identity trust provider issues"
context: "Identity trust providers authenticate users and supply identity claims (email, groups, etc.) for Cedar policy evaluation. Supported types: OIDC and IAM Identity Center. OIDC requires correct issuer, endpoints, client credentials, and scopes. IAM Identity Center integrates natively."
---

## Phase 1 — Triage

MUST:
- Check trust provider config: `aws ec2 describe-verified-access-trust-providers --verified-access-trust-provider-ids <provider-id>`
- Verify OIDC endpoints are accessible (issuer, authorization, token, userinfo)
- Check client ID and secret are correct in IdP
- Verify scopes include required claims

SHOULD:
- Test OIDC token endpoint independently
- Check IdP application configuration (redirect URIs, allowed scopes)
- Verify user exists and is assigned to the application in IdP

MAY:
- Decode JWT tokens to verify claims
- Check IdP logs for authentication errors

## Phase 2 — Remediate

MUST:
- Fix OIDC configuration to match IdP settings
- Ensure redirect URIs in IdP include Verified Access callback URL
- Verify scopes return required claims (email, groups)

SHOULD:
- Test authentication flow end-to-end
- Verify claims are available in Cedar policy context

MAY:
- Configure additional scopes for richer context
- Implement IdP-side access controls as defense in depth

## Common Issues

- symptoms: "Authentication redirect fails"
  diagnosis: "OIDC authorization endpoint incorrect or redirect URI mismatch."
  resolution: "Verify authorization endpoint URL. Add Verified Access callback to IdP redirect URIs."

- symptoms: "Token exchange fails"
  diagnosis: "Client ID/secret incorrect or token endpoint unreachable."
  resolution: "Verify client credentials. Check token endpoint accessibility."

- symptoms: "Claims missing in policy context"
  diagnosis: "Scopes don't include required claims."
  resolution: "Add scopes: openid email profile groups. Verify IdP returns claims."

## Output Format

```yaml
root_cause: "identity_trust_provider — <specific_cause>"
evidence:
  - type: provider_config
    content: "<OIDC/IAM Identity Center configuration>"
  - type: endpoint_test
    content: "<OIDC endpoint accessibility>"
  - type: claims
    content: "<available claims>"
severity: HIGH
mitigation:
  immediate: "Fix IdP configuration and verify authentication flow"
  long_term: "Document IdP setup and test regularly"
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
