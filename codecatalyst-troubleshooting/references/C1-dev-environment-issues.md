---
title: "C1 — Dev Environment Issues"
description: "Diagnose dev environment creation and runtime issues"
status: active
severity: MEDIUM
triggers: ["dev environment failed", "cannot start dev environment", "dev env error", "devfile error"]
owner: devops-agent
objective: "Resolve dev environment creation, startup, and runtime issues"
context: "Dev environments are managed cloud compute instances accessed via IDE. They are NOT EC2 instances."
---

## Phase 1 — Triage

MUST:
- List dev environments: `aws codecatalyst list-dev-environments --space-name <space> --project-name <project>`
- Get environment details: `aws codecatalyst get-dev-environment --space-name <space> --project-name <project> --id <env-id>`
- Check environment status — should be RUNNING

SHOULD:
- Verify devfile.yaml configuration if using custom dev environment
- Check space compute quota usage
- Review environment instance type and storage configuration

MAY:
- Check if the issue affects all dev environments or a specific one
- Verify source repository is accessible from the dev environment

## Phase 2 — Remediate

MUST:
- Fix devfile.yaml syntax errors if environment fails to start
- Ensure space has available compute quota for dev environments
- Verify the selected instance type is available

SHOULD:
- Stop and restart the dev environment if stuck in an error state
- Update devfile.yaml with correct container image references
- Increase storage if environment runs out of disk space

MAY:
- Create a new dev environment if the existing one is unrecoverable
- Configure persistent storage for dev environment data

## Safety Ratings

```
safety_ratings:
  - "list-dev-environments: GREEN — read-only enumeration"
  - "get-dev-environment: GREEN — read-only environment details"
  - "stop-dev-environment: YELLOW — stops running environment"
  - "delete-dev-environment: YELLOW — removes environment and unsaved work"
```

## Escalation Conditions

- All dev environments in the project fail to start
- Dev environment compute quota exhausted for the space
- Dev environment data loss due to unexpected termination
- Dev environment performance severely degraded

## Data Sensitivity

- `aws codecatalyst list-dev-environments`: LOW — returns environment metadata
- `aws codecatalyst get-dev-environment`: LOW — returns environment configuration
- Dev environment content: HIGH — contains developer work in progress
- Devfile configuration: LOW — environment setup definition

## Prohibited Actions

- NEVER delete a dev environment without confirming unsaved work is backed up
- NEVER attempt to SSH into dev environments (they are not EC2 instances)
- NEVER use EC2 APIs to manage dev environments
- NEVER share dev environment access with unauthorized users

## Phase 3 — Rollback

1. If devfile was modified, restore previous version from source control
2. If dev environment was deleted, create a new one with the same configuration
3. Verify source repository is properly cloned in the new environment
4. Confirm IDE connectivity is restored
5. Document the dev environment issue and resolution

## Common Issues

- symptoms: "Dev environment stuck in STARTING status"
  diagnosis: "Devfile may have errors or compute quota may be exhausted."
  resolution: "Check devfile.yaml for errors. Verify space compute quota. Stop and restart the environment."

- symptoms: "Dev environment runs out of storage"
  diagnosis: "Build artifacts or dependencies exceeding storage allocation."
  resolution: "Clean up unnecessary files. Create a new environment with larger storage allocation."

## Output Format

```yaml
root_cause: "dev_environment — <specific_cause>"
evidence:
  - type: environment_status
    content: "<status details>"
severity: MEDIUM
mitigation:
  immediate: "Restore dev environment"
  long_term: "Optimize devfile and storage configuration"
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
