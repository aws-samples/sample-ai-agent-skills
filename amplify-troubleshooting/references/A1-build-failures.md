---
title: "A1 — Build Failures"
description: "Diagnose why Amplify builds are failing"
status: active
severity: HIGH
triggers:
  - "build failed"
  - "build error"
  - "build timeout"
  - "npm install failed"
  - "build command error"
owner: devops-agent
objective: "Identify why Amplify builds are failing and resolve the issue"
context: "Build failures occur due to: incorrect build commands, missing dependencies, incompatible Node.js version, build timeout, memory limits, or syntax errors in application code."
---

## Phase 1 — Triage

MUST:
- Check recent build jobs: `aws amplify list-jobs --app-id <app-id> --branch-name <branch> --max-results 5`
- Get failed job details: `aws amplify get-job --app-id <app-id> --branch-name <branch> --job-id <job-id>`
- Check build settings: `aws amplify get-branch --app-id <app-id> --branch-name <branch> --query 'branch.{BuildSpec:buildSpec,Framework:framework}'`
- Review build logs in Amplify console for specific error messages

SHOULD:
- Check if amplify.yml exists in repository (overrides console settings)
- Verify Node.js version compatibility with the project
- Check if dependencies are properly specified in package.json
- Verify build command matches the framework (npm run build, yarn build, etc.)

MAY:
- Check if build image has required system dependencies
- Review if build cache is causing stale dependency issues
- Check build timeout settings

## Phase 2 — Remediate

MUST:
- Fix the specific build error identified in build logs
- Ensure build commands match the framework requirements
- Verify Node.js version in build settings matches project requirements

SHOULD:
- Set Node.js version in amplify.yml: `frontend: phases: preBuild: commands: - nvm use 18`
- Clear build cache if dependency issues: trigger build with "Clear cache" option
- Increase build timeout if builds are timing out

MAY:
- Use custom build image for specific system dependencies
- Implement build caching optimization in amplify.yml

## Safety Ratings

```
safety_ratings:
  - "list-jobs: GREEN — read-only job status check"
  - "get-job: GREEN — read-only job detail retrieval"
  - "get-branch (buildSpec): GREEN — read-only build config check"
  - "trigger rebuild with cache clear: YELLOW — initiates new build, clears cache"
  - "change Node.js version: YELLOW — changes build environment"
```

## Escalation Conditions

- Build failures blocking production deployments
- Build timeout requires infrastructure-level changes
- Custom build image needed for system dependencies
- Build failures affect multiple branches simultaneously

## Data Sensitivity

- Build logs: MEDIUM — may contain environment variable names and dependency details
- Build settings: LOW — build commands and framework configuration
- amplify.yml: LOW — build configuration file

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER expose secrets or API keys in build logs
- NEVER modify amplify.yml on the main branch without testing on a feature branch first

## Phase 3 — Rollback

1. If build settings were changed and broke the build, restore previous amplify.yml from version control
2. If Node.js version was changed, revert to the previous version in amplify.yml preBuild phase
3. If build cache was cleared and caused issues, trigger a fresh build (cache will rebuild)
4. Revert any build command changes to the last known working configuration
5. Verify the build succeeds with the restored configuration

## Output Format

```yaml
root_cause: "build_failure — <specific_cause>"
evidence:
  - type: build_logs
    content: "<error from build logs>"
  - type: build_config
    content: "<build settings>"
severity: HIGH
mitigation:
  immediate: "Fix the specific build error"
  long_term: "Implement CI checks before Amplify builds"
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
