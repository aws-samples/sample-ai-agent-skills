---
title: "G1 — Blueprint Errors"
description: "Diagnose blueprint creation and application errors"
status: active
severity: MEDIUM
triggers: ["blueprint error", "blueprint failed", "project from blueprint failed", "blueprint not found"]
owner: devops-agent
objective: "Resolve blueprint application and project generation errors"
context: "Blueprints are one-time project generators. They create initial structure but do not continuously manage projects."
---

## Phase 1 — Triage

MUST:
- Verify the blueprint exists and is available in the space
- Check blueprint input parameters for validation errors
- Review project creation logs for specific error messages

SHOULD:
- Verify the user has permission to use blueprints
- Check if the blueprint version is compatible with the space tier
- Review blueprint dependencies (linked repos, AWS connections)

MAY:
- Test with a different blueprint to isolate blueprint-specific issues
- Check if the blueprint has known issues or version updates

## Phase 2 — Remediate

MUST:
- Fix blueprint input parameters (project name, repository name, etc.)
- Ensure required AWS connections are configured before applying blueprint
- Verify space has capacity for new project resources

SHOULD:
- Use latest blueprint version for bug fixes
- Configure all optional parameters for complete project setup
- Review generated project structure after blueprint application

MAY:
- Create custom blueprints for organization-specific project templates
- Modify generated project files to match team standards

## Safety Ratings

```
safety_ratings:
  - "blueprint listing: GREEN — read-only enumeration"
  - "blueprint application: YELLOW — creates project with multiple resources"
  - "project creation from blueprint: YELLOW — generates repos, workflows, environments"
  - "custom blueprint creation: YELLOW — creates reusable project template"
```

## Escalation Conditions

- Blueprint application fails with internal service errors
- Blueprint generates incorrect or broken project structure
- Custom blueprint validation fails during publishing
- Blueprint dependencies cannot be satisfied

## Data Sensitivity

- Blueprint metadata: LOW — template names and descriptions
- Blueprint parameters: LOW — project configuration inputs
- Generated project content: MEDIUM — initial source code and configuration
- Custom blueprint source: MEDIUM — organization-specific templates

## Prohibited Actions

- NEVER assume blueprints continuously manage projects after creation
- NEVER modify blueprint-generated files expecting blueprint to re-sync
- NEVER publish custom blueprints with hardcoded secrets
- NEVER apply blueprints to existing projects (they create new projects)

## Phase 3 — Rollback

1. If a project was created from a blueprint in error, delete the project and its resources
2. If blueprint parameters were incorrect, delete the project and re-apply with correct parameters
3. Verify no orphaned resources remain after project deletion
4. Confirm space quotas are restored after cleanup
5. Document the blueprint error and correct parameters for future use

## Common Issues

- symptoms: "Blueprint application fails with 'invalid parameter' error"
  diagnosis: "Blueprint input parameters do not meet validation requirements."
  resolution: "Review blueprint parameter requirements. Fix project name, repo name, or other inputs."

- symptoms: "Blueprint creates project but workflows fail immediately"
  diagnosis: "Blueprint requires AWS connection that is not configured."
  resolution: "Set up required AWS connection before applying the blueprint. Re-apply if needed."

## Output Format

```yaml
root_cause: "blueprint — <specific_cause>"
evidence:
  - type: blueprint_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix blueprint application"
  long_term: "Document blueprint requirements and validate parameters before application"
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
