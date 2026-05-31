---
title: "A2 — Project Configuration"
description: "Diagnose project configuration issues"
status: active
severity: MEDIUM
triggers: ["project configuration error", "cannot configure project", "project settings wrong"]
owner: devops-agent
objective: "Resolve project configuration and settings issues"
context: "Projects are containers within spaces for source code, workflows, and dev environments."
---

## Phase 1 — Triage

MUST:
- Get project details: `aws codecatalyst get-project --space-name <space> --name <project>`
- List projects in the space: `aws codecatalyst list-projects --space-name <space>`
- Verify user has appropriate project role (Admin, Contributor)

SHOULD:
- Check project source repositories: `aws codecatalyst list-source-repositories --space-name <space> --project-name <project>`
- Review project workflows: `aws codecatalyst list-workflows --space-name <space> --project-name <project>`
- Check project environments

MAY:
- Review CloudTrail for project configuration changes
- Verify linked AWS connections

## Phase 2 — Remediate

MUST:
- Fix project role assignments for affected users
- Verify source repository configuration
- Ensure project has required resources (repos, workflows)

SHOULD:
- Update project description and settings
- Configure project-level environment variables
- Set up branch protection rules

MAY:
- Reorganize project structure using blueprints as reference
- Configure project notifications

## Safety Ratings

```
safety_ratings:
  - "get-project: GREEN — read-only project details"
  - "list-projects: GREEN — read-only project listing"
  - "project configuration changes: YELLOW — modifies project settings"
  - "role assignment changes: YELLOW — modifies access control"
```

## Escalation Conditions

- Project configuration changes do not take effect
- Project becomes inaccessible to all members
- Source repository association is broken
- Project-level settings conflict with space-level policies

## Data Sensitivity

- `aws codecatalyst get-project`: LOW — returns project metadata
- Project member list: MEDIUM — reveals team composition
- Project configuration: LOW — project settings and structure
- Source repository content: HIGH — contains application source code

## Prohibited Actions

- NEVER delete a project without backing up source repositories
- NEVER remove all admin users from a project
- NEVER modify project configuration during active deployments
- NEVER share project access tokens externally

## Phase 3 — Rollback

1. If project settings were modified, restore original configuration
2. If member roles were changed, restore original role assignments
3. Verify all project resources (repos, workflows, environments) are intact
4. Confirm project members can access the project after changes
5. Document configuration changes and their effects

## Common Issues

- symptoms: "User cannot access project despite being a space member"
  diagnosis: "User is not added as a project member or has insufficient role."
  resolution: "Add user to the project with appropriate role (Contributor, Admin)."

- symptoms: "Project settings changes not reflected in workflows"
  diagnosis: "Workflow configuration may override project-level settings."
  resolution: "Check workflow YAML for hardcoded values that override project settings."

## Output Format

```yaml
root_cause: "project_configuration — <specific_cause>"
evidence:
  - type: configuration_details
    content: "<config data>"
severity: MEDIUM
mitigation:
  immediate: "Fix project configuration"
  long_term: "Document project configuration standards"
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
