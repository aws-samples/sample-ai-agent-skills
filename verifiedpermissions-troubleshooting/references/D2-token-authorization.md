---
title: "D2 — Token-Based Authorization"
description: "Debug is-authorized-with-token failures"
status: active
severity: HIGH
triggers:
  - "is-authorized-with-token error"
  - "token authorization failed"
  - "Cognito token rejected"
  - "identity token not recognized"
owner: devops-agent
objective: "Resolve token-based authorization issues with Cognito/OIDC"
context: "is-authorized-with-token accepts JWT tokens from configured identity sources. The token is validated, claims are extracted, and mapped to Cedar entities. The identity source must be configured in the policy store."
---

## Phase 1 — Triage

MUST:
- List identity sources: `aws verifiedpermissions list-identity-sources --policy-store-id <id>`
- Verify identity source config: `aws verifiedpermissions get-identity-source --policy-store-id <id> --identity-source-id <source-id>`
- Test with token: `aws verifiedpermissions is-authorized-with-token --policy-store-id <id> --identity-token <jwt> --action '{"actionType":"MyApp::Action","actionId":"view"}' --resource '{"entityType":"MyApp::Document","entityId":"doc1"}'`
- Decode JWT token to verify claims (use jwt.io or similar)

SHOULD:
- Verify token issuer matches identity source configuration
- Check token expiration
- Verify required claims are present in the token

## Phase 2 — Remediate

MUST:
- Ensure identity source is configured for the correct Cognito user pool or OIDC provider
- Verify token claim-to-entity mapping in identity source configuration
- Fix Cedar policies to reference correct mapped entity types

SHOULD:
- Test with a fresh token (not expired)
- Verify Cognito user pool app client settings

## Common Issues

- symptoms: "Identity source not found for token issuer"
  diagnosis: "No identity source configured for the token's issuer URL."
  resolution: "Create an identity source matching the token issuer."

- symptoms: "Token valid but authorization DENY"
  diagnosis: "Token claims mapped to entity but no permit policy matches."
  resolution: "Check how claims map to Cedar entities and ensure permit policies cover them."


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
