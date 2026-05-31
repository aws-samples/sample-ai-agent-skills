---
title: "F2 — Template-Linked Policies"
description: "Diagnose template-linked policy creation and evaluation issues"
status: active
severity: MEDIUM
triggers:
  - "template-linked policy error"
  - "cannot instantiate template"
  - "template policy not working"
owner: devops-agent
objective: "Resolve template-linked policy instantiation and evaluation problems"
context: "Template-linked policies fill in template placeholders with concrete entity values. Each template-linked policy references a template and provides values for ?principal and ?resource. Values are fixed at creation time."
---

## Phase 1 — Triage

MUST:
- Get the template: `aws verifiedpermissions get-policy-template --policy-store-id <id> --policy-template-id <template-id>`
- List template-linked policies: `aws verifiedpermissions list-policies --policy-store-id <id> --filter '{"policyTemplateId":"<template-id>"}'`
- Verify entity values match schema entity types

SHOULD:
- Test authorization with the template-linked policy's entities
- Check if template was updated after policy was linked

## Phase 2 — Remediate

MUST:
- Delete and recreate template-linked policy if entity values are wrong
- Ensure entity types are fully qualified (Namespace::Type)

SHOULD:
- Automate template-linked policy creation for user onboarding
- Track template-to-policy relationships

## Common Issues

- symptoms: "Template-linked policy creation fails with entity type error"
  diagnosis: "Entity type doesn't match schema or template expectations."
  resolution: "Use fully qualified entity types matching the schema: MyApp::User, MyApp::Document."

- symptoms: "Updating template doesn't update linked policies"
  diagnosis: "Template-linked policies use the current template at evaluation time."
  resolution: "Template changes automatically apply to all linked policies at evaluation time. No re-linking needed."


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
