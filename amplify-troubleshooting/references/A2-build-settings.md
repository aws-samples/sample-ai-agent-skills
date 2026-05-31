---
title: "A2 — Build Settings Issues"
description: "Diagnose build settings not being applied correctly"
status: active
severity: MEDIUM
triggers:
  - "build settings not working"
  - "amplify.yml not applied"
  - "wrong build command"
  - "build configuration"
  - "buildspec issue"
owner: devops-agent
objective: "Identify why build settings are not being applied correctly"
context: "Build settings issues occur when: amplify.yml overrides console settings, build commands are wrong for the framework, base directory is incorrect, or artifacts path is misconfigured."
---

## Phase 1 — Triage

MUST:
- Check if amplify.yml exists in repository root
- Check console build settings: `aws amplify get-branch --app-id <app-id> --branch-name <branch>`
- Compare amplify.yml content with expected build commands
- Verify base directory and artifacts path

SHOULD:
- Check for branch-specific build settings in amplify.yml
- Verify framework detection is correct: `aws amplify get-app --app-id <app-id> --query 'app.{Framework:productionBranch.framework}'`
- Check if monorepo appRoot is set correctly

MAY:
- Review amplify.yml syntax for YAML errors
- Check if build settings differ between branches

## Phase 2 — Remediate

MUST:
- If amplify.yml exists: modify the file, not console settings
- Ensure build commands match framework (e.g., `npm run build` for React)
- Set correct artifacts baseDirectory (e.g., `build` for CRA, `out` for Next.js static)

SHOULD:
- Create proper amplify.yml with all phases (preBuild, build, postBuild)
- Set correct Node.js version in preBuild phase

MAY:
- Use branch-specific configurations in amplify.yml
- Implement build validation in CI before Amplify

## Safety Ratings

```
safety_ratings:
  - "get-branch (build settings): GREEN — read-only config check"
  - "get-app (framework): GREEN — read-only app check"
  - "modify amplify.yml: YELLOW — changes build configuration"
  - "set monorepo appRoot: YELLOW — changes which directory is built"
```

## Escalation Conditions

- Build settings mismatch causing repeated build failures on production branch
- amplify.yml overriding console settings unexpectedly
- Monorepo configuration affecting multiple apps
- Framework auto-detection incorrect, requiring manual override

## Data Sensitivity

- Build settings: LOW — build commands and framework configuration
- amplify.yml content: LOW — build configuration
- Framework detection: LOW — framework metadata

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER modify console build settings when amplify.yml exists (it takes precedence)
- NEVER set incorrect artifacts baseDirectory — it will cause deployment failures

## Phase 3 — Rollback

1. If amplify.yml was modified incorrectly, revert from version control
2. If console build settings were changed, restore previous settings via CLI
3. If monorepo appRoot was changed, restore the correct directory path
4. Trigger a rebuild to verify the restored configuration works
5. Document the correct build settings for the team

## Output Format

```yaml
root_cause: "build_settings_issue — <specific_cause>"
evidence:
  - type: build_config
    content: "<amplify.yml or console settings>"
  - type: framework
    content: "<detected framework>"
severity: MEDIUM
mitigation:
  immediate: "Fix build settings in amplify.yml or console"
  long_term: "Maintain amplify.yml in version control with proper settings"
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
