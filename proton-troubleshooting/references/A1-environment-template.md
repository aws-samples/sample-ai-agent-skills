---
title: "A1 — Proton Environment Template Issues"
description: "Diagnose environment template registration and publishing failures"
status: active
severity: HIGH
triggers:
  - "environment template failed"
  - "template registration error"
  - "REGISTRATION_FAILED"
  - "cannot create environment template"
owner: devops-agent
objective: "Identify why an environment template fails to register or publish and resolve the issue"
context: "Environment templates define shared infrastructure (VPC, clusters, etc.). Template bundles must follow Proton's schema format with schema.yaml, manifest.yaml, and infrastructure templates. Registration validates the bundle; publishing makes it deployable."
---

## Phase 1 — Triage

MUST:
- List environment templates: `aws proton list-environment-templates`
- Check template details: `aws proton get-environment-template --name <template-name>`
- List template versions: `aws proton list-environment-template-versions --template-name <template-name>`
- Check version status: `aws proton get-environment-template-version --template-name <template-name> --major-version <major> --minor-version <minor>`
- Check CloudTrail for registration events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironmentTemplateVersion --max-results 10`

SHOULD:
- Verify template bundle S3 location is accessible
- Check schema.yaml for valid input/output definitions
- Verify manifest.yaml references correct infrastructure files

MAY:
- Download and inspect the template bundle locally
- Validate CloudFormation/Terraform templates independently

## Phase 2 — Remediate

MUST:
- Fix schema.yaml syntax and input/output definitions
- Ensure manifest.yaml correctly references infrastructure template files
- Fix CloudFormation or Terraform template syntax errors
- Re-register the template version after fixes

SHOULD:
- Validate template bundle structure before uploading
- Test CloudFormation templates with cfn-lint before registration
- Use semantic versioning for template versions

MAY:
- Set up template sync for automatic version creation from Git
- Create template validation in CI/CD pipeline

## Common Issues

- symptoms: "Template version status is REGISTRATION_FAILED"
  diagnosis: "Template bundle has invalid schema.yaml, missing manifest.yaml, or template syntax errors."
  resolution: "Check the statusMessage for details. Fix the bundle structure and re-register."

- symptoms: "Cannot publish template version — still in DRAFT"
  diagnosis: "Template version registration succeeded but publish step was not executed."
  resolution: "Publish the version: aws proton update-environment-template-version --template-name <name> --major-version <major> --minor-version <minor> --status PUBLISHED"

- symptoms: "Template version shows as PUBLISHED but deployment fails"
  diagnosis: "Template has valid syntax but runtime errors (invalid resource references, missing parameters)."
  resolution: "Check CloudFormation stack events for the specific deployment error."

## Output Format

```yaml
root_cause: "environment_template_issue — <specific_cause>"
evidence:
  - type: template_version
    content: "<version status and statusMessage>"
  - type: bundle_structure
    content: "<schema.yaml and manifest.yaml details>"
  - type: cloudtrail
    content: "<registration event details>"
severity: HIGH
mitigation:
  immediate: "Fix template bundle and re-register"
  long_term: "Add template validation to CI/CD pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — templates define infrastructure patterns
- Service Disruption Risk: LOW — template issues do not affect running environments
- Compliance Impact: MEDIUM — templates define security baselines for environments

## Escalation Conditions
- Template creation fails with validation errors
- Template incompatible with target infrastructure
- Template version conflicts with existing environments
- Template schema changes break existing deployments
- Multiple templates fail simultaneously

## Data Sensitivity
- Template source code — INTERNAL
- Infrastructure patterns — INTERNAL
- Parameter schemas — INTERNAL
- Template version history — INTERNAL
- Never expose template source that reveals infrastructure architecture

## Prohibited Actions
- Do NOT deploy untested templates to production
- Do NOT modify templates used by active environments without versioning
- Do NOT include credentials in template parameters
- Do NOT bypass template validation
- Do NOT delete templates referenced by active environments
- Do NOT share template source externally

## Phase 3 — Rollback
- Restore previous template versions
- Revert template schema changes
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
