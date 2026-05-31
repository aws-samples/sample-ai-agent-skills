---
title: "G1 — Custom Job Templates"
description: "Diagnose custom job template creation and validation errors"
status: active
severity: MEDIUM
triggers: ["job template error", "template validation failed", "OpenJD error", "custom template not working"]
owner: devops-agent
objective: "Resolve custom job template issues for job submission"
context: "Custom job templates use the Open Job Description (OpenJD) specification to define job structure, steps, environments, and parameters. Template validation occurs at job submission time."
---

## Phase 1 — Triage

MUST:
- Review job template YAML/JSON for syntax errors
- Check template against OpenJD specification requirements
- Verify all referenced parameters are defined in the template

SHOULD:
- Test template with a minimal job submission
- Check step script commands for OS compatibility
- Verify environment definitions are correct

MAY:
- Compare with working template examples
- Review OpenJD specification documentation for supported features

## Phase 2 — Remediate

MUST:
- Fix YAML/JSON syntax errors in the template
- Ensure required OpenJD fields are present (specificationVersion, name, steps)
- Fix step script commands for target OS (Linux bash vs Windows cmd/PowerShell)

SHOULD:
- Add parameter validation rules to catch errors early
- Test templates with dry-run submissions before production use

MAY:
- Create template libraries for common render applications
- Implement template versioning and change management

## Safety Ratings

```
safety_ratings:
  - "Template validation: GREEN — local syntax checking"
  - "create-job with template: YELLOW — submits job for execution"
  - "Template modification: GREEN — local file editing"
  - "Bulk job submission with new template: YELLOW — multiple jobs with cost implications"
```

## Escalation Conditions

- Template requires OpenJD features not supported in current Deadline Cloud version
- Complex multi-step templates with dependencies failing intermittently
- Template changes needed for production pipeline during active rendering
- Cross-platform templates failing on specific OS workers

## Data Sensitivity

- Job templates: MEDIUM — reveal render pipeline configuration and commands
- Template parameters: MEDIUM — may contain file paths and project names
- Script commands: MEDIUM — reveal application invocation and arguments
- Environment variables in templates: HIGH — may contain application paths or credentials

## Prohibited Actions

- NEVER deploy untested templates to production queues
- NEVER include credentials or secrets directly in job templates
- NEVER modify shared templates without notifying all teams using them
- NEVER submit bulk jobs with a new template without single-job testing first

## Phase 3 — Rollback

1. If template change caused job failures, revert to previous template version
2. If jobs were submitted with broken template, cancel them
3. If template parameters were changed, restore previous parameter definitions
4. Resubmit jobs with corrected template after verification

## Common Issues

- symptoms: "Job submission fails with template validation error"
  diagnosis: "Template YAML syntax error or missing required OpenJD fields."
  resolution: "Validate YAML syntax. Ensure specificationVersion, name, and steps fields are present."

- symptoms: "Job runs but step script fails on worker"
  diagnosis: "Script commands incompatible with worker OS or missing application paths."
  resolution: "Check script commands match worker OS. Verify application paths exist on workers."

## Output Format

```yaml
root_cause: "custom_job_template — <specific_cause>"
evidence:
  - type: template_error
    content: "<template content and validation error>"
severity: MEDIUM
mitigation:
  immediate: "Fix template syntax or commands"
  long_term: "Implement template validation in CI/CD pipeline"
```

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
