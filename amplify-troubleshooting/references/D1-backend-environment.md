---
title: "D1 — Backend Environment Issues"
description: "Diagnose Amplify backend environment configuration problems"
status: active
severity: MEDIUM
triggers:
  - "backend environment error"
  - "backend not deploying"
  - "CloudFormation stack error"
  - "backend resources missing"
  - "amplify push failed"
owner: devops-agent
objective: "Identify and resolve backend environment issues"
context: "Backend environment issues include: CloudFormation stack failures, resource provisioning errors, environment not linked to branch, IAM permission issues, or backend resource conflicts."
---

## Phase 1 — Triage

MUST:
- List backend environments: `aws amplify list-backend-environments --app-id <app-id>`
- Check specific environment: `aws amplify get-backend-environment --app-id <app-id> --environment-name <env>`
- Check branch-to-backend mapping: `aws amplify get-branch --app-id <app-id> --branch-name <branch> --query 'branch.backendEnvironmentArn'`
- Check CloudFormation stack status if applicable

SHOULD:
- Verify IAM service role has backend provisioning permissions
- Check for resource naming conflicts
- Review backend deployment logs

MAY:
- Check Amplify CLI version compatibility
- Review team-provider-info.json for environment configuration

## Phase 2 — Remediate

MUST:
- Fix CloudFormation stack errors if backend uses CFN
- Ensure IAM role has required permissions for backend resources
- Link backend environment to correct branch

SHOULD:
- Create backend environment: `aws amplify create-backend-environment --app-id <app-id> --environment-name <env>`
- Update branch with backend: `aws amplify update-branch --app-id <app-id> --branch-name <branch> --backend-environment-arn <arn>`

MAY:
- Delete and recreate failed backend environment
- Use Amplify CLI for backend management

## Safety Ratings

```
safety_ratings:
  - "list-backend-environments: GREEN — read-only enumeration"
  - "get-backend-environment: GREEN — read-only config check"
  - "create-backend-environment: YELLOW — provisions backend resources"
  - "update-branch (backend link): YELLOW — changes branch-to-backend mapping"
```

## Escalation Conditions

- CloudFormation stack failure blocking backend deployment
- Backend resource conflicts between environments
- IAM service role lacks permissions for backend resource provisioning
- Backend environment deletion needed but resources are in use

## Data Sensitivity

- Backend environment details: MEDIUM — reveals backend resource ARNs and configuration
- CloudFormation stack status: MEDIUM — reveals infrastructure provisioning details
- IAM service role: MEDIUM — reveals backend provisioning permissions

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER delete a backend environment that is linked to a production branch
- NEVER modify backend resources directly — use Amplify CLI or IaC

## Phase 3 — Rollback

1. If a backend environment was created incorrectly, delete it (only if not linked to production): `aws amplify delete-backend-environment --app-id <id> --environment-name <env>`
2. If branch-to-backend mapping was changed, restore it: `aws amplify update-branch --app-id <id> --branch-name <branch> --backend-environment-arn <original-arn>`
3. If CloudFormation stack failed, check stack events and fix the root cause before retrying
4. Verify the backend resources are accessible from the application after rollback
5. Document the backend environment configuration for the team

## Output Format

```yaml
root_cause: "backend_environment_issue — <specific_cause>"
evidence:
  - type: backend_config
    content: "<backend environment details>"
  - type: stack_status
    content: "<CloudFormation stack status>"
severity: MEDIUM
mitigation:
  immediate: "Fix backend provisioning error"
  long_term: "Implement backend environment management in CI/CD"
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
