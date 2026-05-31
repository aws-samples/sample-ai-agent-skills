---
title: "F1 — Policy Templates"
description: "Diagnose policy template creation and management issues"
status: active
severity: MEDIUM
triggers:
  - "policy template error"
  - "cannot create template"
  - "template syntax error"
owner: devops-agent
objective: "Resolve policy template creation and configuration problems"
context: "Policy templates define reusable policy patterns with ?principal and ?resource placeholders. Template-linked policies instantiate templates with concrete entity values. Templates enable scalable policy management."
---

## Phase 1 — Triage

MUST:
- List templates: `aws verifiedpermissions list-policy-templates --policy-store-id <id>`
- Get template: `aws verifiedpermissions get-policy-template --policy-store-id <id> --policy-template-id <template-id>`
- Verify template syntax uses ?principal and/or ?resource placeholders
- Check schema compatibility

SHOULD:
- List template-linked policies to see instantiations
- Verify placeholder usage is correct

## Phase 2 — Remediate

MUST:
- Create template: `aws verifiedpermissions create-policy-template --policy-store-id <id> --statement "permit(principal == ?principal, action, resource == ?resource);"`
- Create template-linked policy: `aws verifiedpermissions create-policy --policy-store-id <id> --definition '{"templateLinked":{"policyTemplateId":"<template-id>","principal":{"entityType":"MyApp::User","entityId":"alice"},"resource":{"entityType":"MyApp::Document","entityId":"doc1"}}}'`

SHOULD:
- Test template-linked policies with is-authorized
- Use templates for common access patterns

## Common Issues

- symptoms: "Template creation fails with syntax error"
  diagnosis: "Incorrect placeholder syntax or missing ?principal/?resource."
  resolution: "Use ?principal and ?resource as placeholders. They replace the principal/resource in the scope."

- symptoms: "Template-linked policy doesn't match authorization requests"
  diagnosis: "Entity values in template-linked policy don't match request entities."
  resolution: "Verify entity type and ID in the template-linked policy match the authorization request."


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
