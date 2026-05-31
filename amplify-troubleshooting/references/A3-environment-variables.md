---
title: "A3 — Environment Variable Issues"
description: "Diagnose missing or incorrect environment variables in Amplify builds"
status: active
severity: MEDIUM
triggers:
  - "environment variable missing"
  - "env var not set"
  - "process.env undefined"
  - "API key not found"
  - "configuration missing"
owner: devops-agent
objective: "Identify and resolve environment variable issues"
context: "Environment variable issues include: variable not set, wrong scope (app vs branch), variable not available at runtime for static sites, reserved variable names, or secrets not properly configured."
---

## Phase 1 — Triage

MUST:
- Check app-level environment variables: `aws amplify get-app --app-id <app-id> --query 'app.environmentVariables'`
- Check branch-level environment variables: `aws amplify get-branch --app-id <app-id> --branch-name <branch> --query 'branch.environmentVariables'`
- Verify the variable is being used correctly in build/runtime context
- Check if the app is static (WEB) or SSR (WEB_COMPUTE)

SHOULD:
- Verify variable naming convention matches framework (REACT_APP_*, NEXT_PUBLIC_*, VUE_APP_*)
- Check if branch-level vars are overriding app-level vars
- Verify secrets are not being logged in build output

MAY:
- Check if amplify.yml references environment variables correctly
- Review if _LIVE_UPDATES or other reserved names conflict

## Phase 2 — Remediate

MUST:
- Set missing variables at appropriate scope (app or branch level)
- For static sites: ensure variables are embedded during build, not expected at runtime
- Use correct framework prefix for client-side variables

SHOULD:
- Set environment variable: `aws amplify update-branch --app-id <app-id> --branch-name <branch> --environment-variables KEY=value`
- Trigger rebuild after adding variables

MAY:
- Use AWS Systems Manager Parameter Store for sensitive values
- Implement .env.example file for documentation

## Safety Ratings

```
safety_ratings:
  - "get-app (environmentVariables): GREEN — read-only env var check"
  - "get-branch (environmentVariables): GREEN — read-only env var check"
  - "update-branch (set env var): YELLOW — changes build/runtime configuration"
  - "trigger rebuild after env var change: YELLOW — initiates new build"
```

## Escalation Conditions

- Missing environment variables causing production application errors
- Secrets accidentally exposed in build logs
- Environment variable changes needed across multiple branches
- Reserved variable names conflicting with application variables

## Data Sensitivity

- Environment variable names: LOW — variable names are not sensitive
- Environment variable values: HIGH — may contain API keys, secrets, and credentials
- Build logs with env vars: HIGH — may inadvertently expose secret values

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER log or display environment variable values in troubleshooting output
- NEVER set secrets as plain text in amplify.yml (use console or SSM Parameter Store)

## Phase 3 — Rollback

1. If environment variables were changed incorrectly, restore previous values: `aws amplify update-branch --app-id <id> --branch-name <branch> --environment-variables KEY=previous-value`
2. If a secret was accidentally exposed, rotate the secret immediately
3. Trigger a rebuild after restoring environment variables
4. Verify the application functions correctly with the restored variables
5. Review build logs to ensure no secrets are exposed

## Output Format

```yaml
root_cause: "env_var_issue — <specific_cause>"
evidence:
  - type: env_vars
    content: "<configured variables>"
  - type: app_platform
    content: "<WEB or WEB_COMPUTE>"
severity: MEDIUM
mitigation:
  immediate: "Set missing environment variables and rebuild"
  long_term: "Document required variables and implement validation"
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
