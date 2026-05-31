---
title: "B1 — Cedar Policy Authoring"
description: "Diagnose Cedar policy creation and syntax issues"
status: active
severity: HIGH
triggers:
  - "Cedar policy error"
  - "policy syntax error"
  - "cannot create policy"
  - "permit forbid syntax"
owner: devops-agent
objective: "Resolve Cedar policy authoring and creation problems"
context: "Cedar policies use permit/forbid statements with when/unless clauses. Policies reference entity types defined in the schema. Static policies have concrete principals/resources; template-linked policies use placeholders."
---

## Phase 1 — Triage

MUST:
- List policies: `aws verifiedpermissions list-policies --policy-store-id <id>`
- Get specific policy: `aws verifiedpermissions get-policy --policy-store-id <id> --policy-id <policy-id>`
- Check schema for valid entity types: `aws verifiedpermissions get-schema --policy-store-id <id>`
- Verify Cedar syntax: permit/forbid keywords, when/unless clauses, semicolons

SHOULD:
- Validate entity type namespaces match schema (Namespace::Type)
- Check action names match schema definitions
- Verify attribute references exist in entity type definitions

## Phase 2 — Remediate

MUST:
- Create static policy: `aws verifiedpermissions create-policy --policy-store-id <id> --definition '{"static":{"statement":"permit(principal,action,resource) when { principal.role == \"admin\" };"}}'`
- Verify policy creation: `aws verifiedpermissions get-policy --policy-store-id <id> --policy-id <new-id>`

SHOULD:
- Test policies with is-authorized before deploying
- Use descriptive policy descriptions

## Common Issues

- symptoms: "ValidationException: policy syntax error"
  diagnosis: "Missing semicolon, incorrect keyword, or malformed when clause."
  resolution: "Check Cedar syntax: statements end with semicolons, use == for equality, .contains() for sets."

- symptoms: "Policy created but authorization always denies"
  diagnosis: "Entity type mismatch between policy and authorization request."
  resolution: "Ensure principal/resource types in policy match those in is-authorized calls."


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
