---
title: "B2 — Branch Configuration Issues"
description: "Diagnose branch configuration and auto-detection problems"
status: active
severity: MEDIUM
triggers:
  - "branch not deploying"
  - "branch not detected"
  - "wrong branch settings"
  - "branch auto-detection"
  - "branch pattern"
owner: devops-agent
objective: "Identify and resolve branch configuration issues"
context: "Branch issues include: auto-detection not finding branches, branch patterns excluding desired branches, wrong stage assignment, or branch-specific settings not applied."
---

## Phase 1 — Triage

MUST:
- List all branches: `aws amplify list-branches --app-id <app-id>`
- Check branch configuration: `aws amplify get-branch --app-id <app-id> --branch-name <branch>`
- Check auto-branch creation settings: `aws amplify get-app --app-id <app-id> --query 'app.autoBranchCreationConfig'`
- Verify repository connection: `aws amplify get-app --app-id <app-id> --query 'app.{Repo:repository,Provider:repositoryCloneMethod}'`

SHOULD:
- Check branch auto-detection patterns (include/exclude)
- Verify webhook is configured in repository provider
- Check if branch exists in the repository

MAY:
- Review branch creation history in CloudTrail
- Check for branch name conflicts or special characters

## Phase 2 — Remediate

MUST:
- If branch not detected: verify auto-branch creation patterns
- If webhook missing: reconnect repository or manually create webhook
- Create branch manually if auto-detection fails: `aws amplify create-branch --app-id <app-id> --branch-name <branch>`

SHOULD:
- Update auto-branch patterns: `aws amplify update-app --app-id <app-id> --auto-branch-creation-patterns '["feature/*","develop"]'`
- Set correct stage for branch (PRODUCTION, BETA, DEVELOPMENT, PULL_REQUEST)

MAY:
- Configure branch-specific environment variables
- Set up branch-specific build settings

## Safety Ratings

```
safety_ratings:
  - "list-branches: GREEN — read-only branch enumeration"
  - "get-branch: GREEN — read-only branch config check"
  - "get-app (autoBranchCreationConfig): GREEN — read-only auto-detection check"
  - "create-branch: YELLOW — creates new branch deployment"
  - "update-app (auto-branch patterns): YELLOW — changes auto-detection behavior"
```

## Escalation Conditions

- Production branch not deploying due to configuration issues
- Webhook misconfiguration affecting all branches
- Auto-branch creation patterns too broad, creating unwanted deployments
- Repository connection lost, requiring re-authentication

## Data Sensitivity

- Branch configuration: LOW — branch settings and stage
- Auto-branch patterns: LOW — branch naming patterns
- Repository connection: MEDIUM — reveals source code repository details

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER set auto-branch patterns that match all branches without review
- NEVER disconnect the repository without understanding the impact on all branches

## Phase 3 — Rollback

1. If a branch was created incorrectly, delete it: `aws amplify delete-branch --app-id <id> --branch-name <branch>` (only non-production)
2. If auto-branch patterns were changed, restore previous patterns: `aws amplify update-app --app-id <id> --auto-branch-creation-patterns '<original-patterns>'`
3. If webhook was modified, restore it in the repository provider settings
4. Verify all expected branches are deploying correctly after rollback
5. Document the branch configuration changes

## Output Format

```yaml
root_cause: "branch_config_issue — <specific_cause>"
evidence:
  - type: branch_config
    content: "<branch settings>"
  - type: auto_detection
    content: "<auto-branch patterns>"
severity: MEDIUM
mitigation:
  immediate: "Fix branch configuration or create branch manually"
  long_term: "Configure proper auto-branch patterns and webhook"
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
