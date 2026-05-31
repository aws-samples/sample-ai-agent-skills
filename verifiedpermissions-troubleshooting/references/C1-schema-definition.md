---
title: "C1 — Schema Definition"
description: "Diagnose schema creation and entity type definition issues"
status: active
severity: HIGH
triggers:
  - "schema error"
  - "entity type definition"
  - "cannot create schema"
  - "put-schema failed"
owner: devops-agent
objective: "Resolve schema definition and creation problems"
context: "The schema defines entity types, actions, and context for Cedar policies. It uses Cedar schema format (JSON). Entity types have attributes with types. Actions specify applicable principal and resource types."
---

## Phase 1 — Triage

MUST:
- Get current schema: `aws verifiedpermissions get-schema --policy-store-id <id>`
- Check if schema exists (empty response means no schema)
- Verify schema JSON syntax
- Check entity type definitions for correct attribute types

SHOULD:
- Verify action definitions reference valid entity types
- Check parent-child entity type relationships
- Validate attribute types (String, Long, Boolean, Set, Record, Entity)

## Phase 2 — Remediate

MUST:
- Create/update schema: `aws verifiedpermissions put-schema --policy-store-id <id> --definition '{"cedarJson":"{\"MyApp\":{\"entityTypes\":{\"User\":{\"shape\":{\"type\":\"Record\",\"attributes\":{\"role\":{\"type\":\"String\"}}}}},\"actions\":{\"viewDocument\":{\"appliesTo\":{\"principalTypes\":[\"User\"],\"resourceTypes\":[\"Document\"]}}}}}"}'`

SHOULD:
- Define all entity types before creating policies
- Include all attributes referenced in policies

## Common Issues

- symptoms: "Invalid schema JSON"
  diagnosis: "Malformed Cedar schema JSON or incorrect nesting."
  resolution: "Validate JSON syntax. Schema uses nested namespace → entityTypes/actions structure."

- symptoms: "Schema update breaks existing policies"
  diagnosis: "Removed entity type or action still referenced by policies."
  resolution: "Update or delete policies referencing removed types before updating schema."


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
