---
title: "A2 — App Runner Build Failures"
description: "Diagnose build failures for code-based App Runner services"
status: active
severity: HIGH
triggers:
  - "build failed"
  - "build error"
  - "build timeout"
  - "compilation error"
  - "dependency install failed"
owner: devops-agent
objective: "Identify why the App Runner build phase failed and fix the build configuration"
context: "Build failures only apply to code-based sources (GitHub/Bitbucket). ECR image sources skip the build phase entirely. App Runner builds the container using the configured runtime and build commands. Common causes include incorrect build commands, missing dependencies, build timeout, and incompatible runtime versions."
---

## Phase 1 — Triage

MUST:
- Confirm the service uses a code source: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.CodeRepository'`
- Check build logs in CloudWatch: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/service --log-stream-name <stream>`
- Verify build configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.CodeRepository.CodeConfiguration'`
- Check the configured runtime and build/start commands
- List recent operations for build-specific errors: `aws apprunner list-operations --service-arn <service-arn>`

SHOULD:
- Verify the source branch exists and has the expected code
- Check if apprunner.yaml exists in the repository root (if using repository configuration)
- Verify build environment variables are correctly set

MAY:
- Test the build locally using the same runtime and commands
- Check if the build exceeds the 10-minute default timeout

## Phase 2 — Remediate

MUST:
- Fix build command errors identified in the build logs
- Ensure all dependencies are available and compatible with the configured runtime
- Verify the apprunner.yaml or API-provided build configuration matches the project structure

SHOULD:
- Use specific dependency versions to avoid build inconsistencies
- Set build environment variables for any build-time configuration (API keys for private registries, etc.)
- Increase build timeout if the build legitimately takes longer than 10 minutes

MAY:
- Switch to ECR image source if build issues persist (pre-build locally and push to ECR)
- Add a build validation step in CI/CD before pushing to the source branch

## Common Issues

- symptoms: "Build fails with 'command not found'"
  diagnosis: "Build command references a tool not available in the selected runtime."
  resolution: "Check the App Runner managed runtime documentation for available tools. Install missing tools in the build command."

- symptoms: "Build times out after 10 minutes"
  diagnosis: "Build process exceeds the default timeout. Large dependency installs or compilation."
  resolution: "Optimize build steps. Use dependency caching. Consider pre-building and using ECR image source."

- symptoms: "Build fails with dependency resolution errors"
  diagnosis: "Package manager cannot resolve dependencies — version conflicts or private registry access."
  resolution: "Pin dependency versions. Configure build environment variables for private registry authentication."

## Output Format

```yaml
root_cause: "build_failure — <specific_cause>"
evidence:
  - type: build_logs
    content: "<build log error output>"
  - type: build_configuration
    content: "<runtime, build command, start command>"
  - type: source_config
    content: "<repository URL and branch>"
severity: HIGH
mitigation:
  immediate: "Fix build configuration and trigger new deployment"
  long_term: "Add local build validation and consider ECR image source for complex builds"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix build command errors | GREEN | Code/config fix |
| Ensure dependencies are available | GREEN | Dependency management |
| Verify apprunner.yaml configuration | GREEN | Config verification |
| Set build environment variables | YELLOW | Environment configuration change |
| Increase build timeout | GREEN | Timeout adjustment |
| Switch to ECR image source | YELLOW | Source type change |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Build logs**: Compilation output, dependency info — INTERNAL
- **Build configuration**: Runtime, commands, env vars — INTERNAL
- **Source repository**: Code repository URL and branch — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER expose private registry credentials in build logs

## Phase 3 — Rollback

1. Revert build configuration changes (apprunner.yaml or API config)
2. Revert source branch to previous known-good commit
3. Trigger redeployment with the reverted configuration
4. Verify build succeeds and service is healthy
5. Remove any temporary build environment variables

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
