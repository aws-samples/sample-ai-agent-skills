---
title: "C2 — Schema Validation"
description: "Diagnose schema validation and type checking issues"
status: active
severity: MEDIUM
triggers:
  - "schema validation error"
  - "type mismatch"
  - "attribute not found in schema"
owner: devops-agent
objective: "Resolve schema validation errors for policies and authorization requests"
context: "Schema validation ensures policies reference valid entity types, actions, and attributes. In STRICT mode, all policies must conform. Authorization requests should use entity types defined in the schema."
---

## Phase 1 — Triage

MUST:
- Get schema: `aws verifiedpermissions get-schema --policy-store-id <id>`
- Check validation mode: `aws verifiedpermissions get-policy-store --policy-store-id <id>`
- List policies with validation errors: attempt re-creation of suspect policies

SHOULD:
- Compare policy entity types against schema definitions
- Verify attribute types match (String vs Long vs Boolean)
- Check action appliesTo constraints

## Phase 2 — Remediate

MUST:
- Fix type mismatches in policies to match schema
- Add missing entity types or attributes to schema
- Ensure action appliesTo includes correct principal and resource types

SHOULD:
- Test authorization with sample requests after schema changes
- Document schema changes and their impact on policies

## Common Issues

- symptoms: "Action not applicable to principal type"
  diagnosis: "Action's appliesTo.principalTypes doesn't include the principal type used."
  resolution: "Add the principal type to the action's appliesTo.principalTypes in the schema."

- symptoms: "Attribute type mismatch in when clause"
  diagnosis: "Policy compares attribute as String but schema defines it as Long."
  resolution: "Fix the comparison or update the schema attribute type."


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
