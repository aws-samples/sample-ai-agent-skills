---
title: "C1 — npm Issues"
description: "Diagnose npm package resolution and installation failures with CodeArtifact"
status: active
severity: HIGH
triggers:
  - "npm install failed"
  - "npm codeartifact"
  - "npm registry error"
  - "npm 401"
  - "npm package not found"
owner: devops-agent
objective: "Resolve npm integration issues with CodeArtifact"
context: "npm requires explicit registry configuration to use CodeArtifact. Common issues include missing registry config, expired auth tokens, scope misconfiguration, and .npmrc conflicts."
---

## Phase 1 — Triage

MUST:
- Check npm registry config: `npm config get registry`
- Check for scoped registries: `npm config list`
- Verify .npmrc file contents: check project and user-level .npmrc
- Get CodeArtifact endpoint: `aws codeartifact get-repository-endpoint --domain <domain> --repository <repo> --format npm`
- Test auth: `npm whoami --registry <endpoint>`

SHOULD:
- Check if token is expired (12-hour max)
- Verify package exists in repository: `aws codeartifact list-packages --domain <domain> --repository <repo> --format npm`
- Check upstream connections for public npm packages

MAY:
- Run with verbose logging: `npm install --verbose`
- Check for .npmrc in parent directories that may override settings

## Phase 2 — Remediate

MUST:
- Configure npm with CodeArtifact: `aws codeartifact login --tool npm --domain <domain> --repository <repo>`
- If scoped packages, configure per-scope: `npm config set @scope:registry <endpoint>`
- Refresh expired token before install

SHOULD:
- Use project-level .npmrc for team consistency
- Add `aws codeartifact login` to CI/CD pipeline before npm install

## Output Format

```yaml
root_cause: "npm_issue — <specific_cause>"
evidence:
  - type: npm_config
    content: "<npm registry configuration>"
  - type: auth_status
    content: "<npm whoami result>"
severity: HIGH
mitigation:
  immediate: "Configure npm registry and refresh auth token"
  long_term: "Use project .npmrc and automate token refresh in CI/CD"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Configure npm with CodeArtifact login | GREEN |
| Configure per-scope registry | GREEN |
| Refresh expired token | GREEN |
| Use project-level .npmrc | GREEN |
| Add codeartifact login to CI/CD pipeline | GREEN |

## Escalation Conditions

- Fix requires modifying production CI/CD pipeline npm configurations
- Fix involves changing .npmrc that affects all team members

## Data Sensitivity

- **MEDIUM**: npm configurations contain registry endpoints and auth tokens
- .npmrc files may be committed to source control exposing registry details

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If .npmrc was modified, restore previous version from source control
2. If npm registry was changed, restore previous registry: `npm config set registry <previous-registry>`
3. If scoped registry was added, remove it: `npm config delete @scope:registry`
4. Verify rollback by confirming `npm install` resolves packages correctly

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
