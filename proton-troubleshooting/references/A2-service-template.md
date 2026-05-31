---
title: "A2 — Proton Service Template Issues"
description: "Diagnose service template registration and publishing failures"
status: active
severity: HIGH
triggers:
  - "service template failed"
  - "service template registration"
  - "service template error"
  - "pipeline template"
owner: devops-agent
objective: "Identify why a service template fails to register or publish and resolve the issue"
context: "Service templates define per-service infrastructure and optionally CI/CD pipelines. They reference environment template outputs as inputs. Template bundles include infrastructure/ and optionally pipeline/ directories."
---

## Phase 1 — Triage

MUST:
- List service templates: `aws proton list-service-templates`
- Check template details: `aws proton get-service-template --name <template-name>`
- List template versions: `aws proton list-service-template-versions --template-name <template-name>`
- Check version status and message: `aws proton get-service-template-version --template-name <template-name> --major-version <major> --minor-version <minor>`
- Verify compatible environment templates: check the version's compatibleEnvironmentTemplates field

SHOULD:
- Verify schema.yaml input definitions match environment outputs
- Check pipeline/ directory if pipeline is expected
- Validate infrastructure templates independently

MAY:
- Compare with a working service template version
- Check if the template references deprecated resource types

## Phase 2 — Remediate

MUST:
- Fix schema.yaml to correctly reference environment output types
- Ensure compatible environment templates are specified
- Fix infrastructure and pipeline template syntax
- Re-register after fixes

SHOULD:
- Test service template against a staging environment
- Validate pipeline configuration if included
- Document input/output contracts between environment and service templates

MAY:
- Use template sync for automatic version management
- Create integration tests for template deployments

## Common Issues

- symptoms: "Service template REGISTRATION_FAILED"
  diagnosis: "Bundle structure invalid or schema references non-existent environment outputs."
  resolution: "Check statusMessage, fix schema.yaml and bundle structure, re-register."

- symptoms: "Service template published but service creation fails with input error"
  diagnosis: "Service template inputs don't match environment outputs."
  resolution: "Verify environment outputs match service template input schema."

- symptoms: "Pipeline section missing from service template"
  diagnosis: "Template bundle doesn't include pipeline/ directory."
  resolution: "Add pipeline/ directory with manifest.yaml and pipeline template if CI/CD is needed."

## Output Format

```yaml
root_cause: "service_template_issue — <specific_cause>"
evidence:
  - type: template_version
    content: "<version status and statusMessage>"
  - type: schema
    content: "<schema.yaml input/output definitions>"
  - type: compatibility
    content: "<compatible environment templates>"
severity: HIGH
mitigation:
  immediate: "Fix template bundle and re-register"
  long_term: "Implement template contract testing between environment and service templates"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — service templates define deployment patterns
- Service Disruption Risk: LOW — template issues do not affect running services
- Compliance Impact: MEDIUM — templates define security baselines for services

## Escalation Conditions
- Service template creation fails
- Template incompatible with target environment
- Template version conflicts with existing services
- Template changes break existing service deployments
- Pipeline template configuration errors

## Data Sensitivity
- Template source code — INTERNAL
- Service deployment patterns — INTERNAL
- Pipeline configurations — INTERNAL
- Template parameters — INTERNAL
- Never expose template source that reveals service architecture

## Prohibited Actions
- Do NOT deploy untested service templates
- Do NOT modify templates used by active services without versioning
- Do NOT include credentials in template parameters
- Do NOT bypass template validation
- Do NOT delete templates referenced by active services
- Do NOT share template source externally

## Phase 3 — Rollback
- Restore previous service template versions
- Revert template changes
- Verify template compatibility after changes
- Restore template parameter defaults
- Document template issues and version history
- Update template management procedures

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
