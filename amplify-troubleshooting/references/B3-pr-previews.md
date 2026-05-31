---
title: "B3 — PR Preview Issues"
description: "Diagnose pull request preview deployment problems"
status: active
severity: LOW
triggers:
  - "PR preview not working"
  - "pull request preview"
  - "preview not deploying"
  - "preview URL not available"
  - "PR build not triggering"
owner: devops-agent
objective: "Identify and resolve PR preview issues"
context: "PR preview issues include: previews not enabled, webhook not triggering, preview URL not accessible, environment variables missing in preview, or previews not cleaning up after PR close."
---

## Phase 1 — Triage

MUST:
- Check if PR previews are enabled: `aws amplify get-app --app-id <app-id> --query 'app.enableAutoBranchCreation'`
- Check auto-branch creation config: `aws amplify get-app --app-id <app-id> --query 'app.autoBranchCreationConfig'`
- Verify PR preview setting on the app: `aws amplify get-app --app-id <app-id> --query 'app.enableBranchAutoBuild'`
- Check if PR branch was created: `aws amplify list-branches --app-id <app-id>`

SHOULD:
- Verify webhook is configured in repository for PR events
- Check if PR preview pattern matches PR branch naming
- Verify build succeeded for the PR branch

MAY:
- Check if PR preview URL is accessible
- Review PR preview environment variables

## Phase 2 — Remediate

MUST:
- Enable PR previews: `aws amplify update-app --app-id <app-id> --enable-auto-branch-creation --auto-branch-creation-config '{"enablePullRequestPreview":true}'`
- Verify webhook includes PR events
- Check repository provider webhook configuration

SHOULD:
- Configure PR preview settings (auto-disconnect, environment variables)
- Ensure build settings work for PR branches

MAY:
- Set up PR preview-specific environment variables
- Configure access control for PR previews

## Safety Ratings

```
safety_ratings:
  - "get-app (enableAutoBranchCreation): GREEN — read-only config check"
  - "list-branches: GREEN — read-only branch enumeration"
  - "update-app (enable PR previews): YELLOW — enables automatic deployments for PRs"
  - "configure PR preview settings: YELLOW — changes preview behavior"
```

## Escalation Conditions

- PR previews consuming excessive build minutes
- PR preview URLs accessible without authentication for sensitive applications
- Webhook configuration requires repository admin access
- PR previews not cleaning up after PR close, leaving orphaned deployments

## Data Sensitivity

- PR preview configuration: LOW — preview settings
- Webhook configuration: MEDIUM — reveals repository integration details
- PR preview URLs: MEDIUM — may expose pre-release features

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER enable PR previews without considering build minute costs
- NEVER expose PR previews for applications with sensitive data without access control

## Phase 3 — Rollback

1. If PR previews were enabled incorrectly, disable them: `aws amplify update-app --app-id <id> --no-enable-auto-branch-creation`
2. Delete orphaned PR branches: `aws amplify delete-branch --app-id <id> --branch-name <pr-branch>`
3. Restore webhook configuration in the repository provider
4. Verify production branches are unaffected by PR preview changes
5. Document the PR preview configuration for the team

## Output Format

```yaml
root_cause: "pr_preview_issue — <specific_cause>"
evidence:
  - type: preview_config
    content: "<PR preview settings>"
  - type: webhook
    content: "<webhook configuration>"
severity: LOW
mitigation:
  immediate: "Enable PR previews and verify webhook"
  long_term: "Configure comprehensive PR preview settings"
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
