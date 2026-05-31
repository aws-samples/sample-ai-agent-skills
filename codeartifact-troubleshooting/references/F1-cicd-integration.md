---
title: "F1 — CI/CD Integration Issues"
description: "Diagnose CodeArtifact integration problems with CI/CD pipelines including CodeBuild, GitHub Actions, and Jenkins"
status: active
severity: HIGH
triggers:
  - "cicd codeartifact"
  - "codebuild codeartifact"
  - "github actions codeartifact"
  - "pipeline auth failed"
  - "build cannot resolve dependencies"
owner: devops-agent
objective: "Resolve CI/CD pipeline integration issues with CodeArtifact"
context: "CI/CD pipelines need to authenticate to CodeArtifact before package operations. Common issues include token expiry during builds, missing IAM permissions on build roles, and incorrect environment variable configuration."
---

## Phase 1 — Triage

MUST:
- Verify build role has CodeArtifact permissions: `codeartifact:GetAuthorizationToken`, `codeartifact:GetRepositoryEndpoint`, `codeartifact:ReadFromRepository`, `sts:GetServiceBearerToken`
- Check if token refresh is in the build script before package operations
- Verify the correct domain and repository are referenced
- Check build logs for auth errors (401, 403)

SHOULD:
- Verify build role trust policy allows the CI/CD service
- Check if build duration exceeds 12 hours (token expiry)
- Verify environment variables are set correctly

MAY:
- Check CodeBuild project environment configuration
- Verify GitHub Actions OIDC role configuration
- Check Jenkins credential store

## Phase 2 — Remediate

MUST:
- Add token refresh step before package operations in build script:
  ```
  aws codeartifact login --tool <npm|pip|mvn> --domain <domain> --repository <repo>
  ```
- Grant required IAM permissions to build role
- Ensure `sts:GetServiceBearerToken` is in the build role policy

SHOULD:
- Use `aws codeartifact login` instead of manual token configuration
- Place token refresh immediately before package install/build step

## Output Format

```yaml
root_cause: "cicd_integration — <specific_cause>"
evidence:
  - type: build_role
    content: "<IAM role permissions>"
  - type: build_logs
    content: "<relevant error logs>"
severity: HIGH
mitigation:
  immediate: "Fix build role permissions and add token refresh"
  long_term: "Standardize CodeArtifact auth in CI/CD templates"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Add token refresh step in build script | GREEN |
| Grant required IAM permissions to build role | YELLOW |
| Ensure sts:GetServiceBearerToken in build role | YELLOW |
| Use aws codeartifact login instead of manual config | GREEN |
| Place token refresh before package install step | GREEN |

## Escalation Conditions

- Fix requires modifying production CI/CD pipeline configurations
- Fix involves changing build role IAM permissions

## Data Sensitivity

- **HIGH**: CI/CD integration configurations contain build role credentials and repository access
- Build logs may expose auth tokens and package resolution details

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If build role permissions were modified, revert to previous IAM policy
2. If build script was changed, restore previous version from source control
3. If environment variables were modified, restore previous values
4. Verify rollback by confirming CI/CD pipeline builds complete successfully

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
