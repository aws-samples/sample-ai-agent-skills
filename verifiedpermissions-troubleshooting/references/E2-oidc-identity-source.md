---
title: "E2 — OIDC Identity Source"
description: "Diagnose OIDC provider identity source integration"
status: active
severity: MEDIUM
triggers:
  - "OIDC identity source"
  - "OIDC provider integration"
  - "external IdP token"
owner: devops-agent
objective: "Resolve OIDC identity source configuration issues"
context: "OIDC identity sources connect external identity providers to Verified Permissions. Tokens from the OIDC provider are validated and claims mapped to Cedar entities. The issuer URL must match the OIDC discovery endpoint."
---

## Phase 1 — Triage

MUST:
- List identity sources: `aws verifiedpermissions list-identity-sources --policy-store-id <id>`
- Get OIDC identity source config: `aws verifiedpermissions get-identity-source --policy-store-id <id> --identity-source-id <source-id>`
- Verify OIDC issuer URL is accessible and returns valid discovery document
- Check token claims match expected mapping

SHOULD:
- Verify OIDC provider's JWKS endpoint is accessible
- Check audience (aud) claim matches configuration
- Verify token signing algorithm is supported

## Phase 2 — Remediate

MUST:
- Create OIDC identity source with correct issuer and entity type mapping
- Verify token validation works with is-authorized-with-token
- Ensure Cedar policies reference correct mapped entity types

SHOULD:
- Test with tokens from different OIDC scopes
- Document claim-to-entity mapping

## Common Issues

- symptoms: "Token validation failed — issuer mismatch"
  diagnosis: "Token issuer doesn't match identity source issuer URL."
  resolution: "Ensure identity source issuer URL exactly matches the iss claim in tokens."

- symptoms: "Claims not available in Cedar policy"
  diagnosis: "Token claims not included in the configured mapping."
  resolution: "Update identity source configuration to map required claims to entity attributes."


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

- Policy stores contain complete authorization logic
- Schema definitions reveal application data model
- Authorization decisions log access patterns
- Classification: CONFIDENTIAL - authorization policies and access decision data

## Prohibited Actions

- NEVER delete policy stores used by production applications
- NEVER modify Cedar policies without testing authorization decisions first
- NEVER change schema definitions that could invalidate existing policies
- NEVER remove identity sources connected to active authentication flows

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
