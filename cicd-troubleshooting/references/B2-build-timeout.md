---
title: "B2 — Build Timeout"
description: "Diagnose and resolve CodeBuild timeout issues"
status: active
severity: MEDIUM
triggers:
  - "build timeout"
  - "timed out"
  - "TIMED_OUT"
  - "build too slow"
  - "60 minutes"
owner: devops-agent
objective: "Identify why the build is timing out and optimize or increase the timeout"
context: "CodeBuild default timeout is 60 minutes, maximum is 8 hours (480 minutes). Timeout applies to active build time only — queue time is excluded. Common causes include large Docker builds, extensive test suites, slow dependency downloads, and inefficient build scripts."
---

## Phase 1 — Triage

MUST:
- Get build details and check timeout: `aws codebuild batch-get-builds --ids <build-id>`
- Check project timeout configuration: `aws codebuild batch-get-projects --names <project-name> --query 'projects[0].timeoutInMinutes'`
- Identify which phase consumed the most time: check `phases[].durationInSeconds` in build details
- Check build logs for the last activity before timeout

SHOULD:
- Compare build duration across recent builds for trends: `aws codebuild list-builds-for-project --project-name <project-name> --max-items 10`
- Check if caching is configured: `aws codebuild batch-get-projects --names <project-name> --query 'projects[0].cache'`
- Check compute type — smaller instances build slower: `aws codebuild batch-get-projects --names <project-name> --query 'projects[0].environment.computeType'`

MAY:
- Check if the build is stuck waiting for a network resource (VPC configuration)
- Review Docker build steps for inefficient layer ordering
- Check if test parallelization is possible

## Phase 2 — Remediate

MUST:
- If timeout is too low for the build: increase `timeoutInMinutes` in project configuration (max 480)
- If a specific phase is slow: optimize that phase (see common issues below)
- If build is stuck (not progressing): check VPC/network configuration, dependency mirrors

SHOULD:
- Enable caching (S3 or local) to speed up dependency downloads
- Use a larger compute type for CPU/memory-intensive builds
- Optimize Docker builds: order layers by change frequency, use multi-stage builds, leverage cache

MAY:
- Split large builds into multiple smaller CodeBuild projects
- Use parallel test execution to reduce test phase duration
- Pre-bake dependencies into a custom build image

## Common Issues

- symptoms: "Build TIMED_OUT at 60 minutes during BUILD phase"
  diagnosis: "Default timeout of 60 minutes is insufficient for the build."
  resolution: "Increase timeoutInMinutes: `aws codebuild update-project --name <project> --timeout-in-minutes 120`."

- symptoms: "INSTALL phase takes 15+ minutes downloading dependencies"
  diagnosis: "Dependencies are downloaded fresh every build. No caching configured."
  resolution: "Enable S3 caching for dependency directories (node_modules, .m2, pip cache)."

- symptoms: "Docker build takes 30+ minutes"
  diagnosis: "Docker layers are not cached, or Dockerfile has inefficient layer ordering."
  resolution: "Enable DOCKER_LAYER_CACHE (requires privileged mode). Reorder Dockerfile to put rarely-changing layers first."

- symptoms: "Build stuck at PROVISIONING for extended time"
  diagnosis: "CodeBuild is waiting for build capacity. This is queue time, not build time."
  resolution: "Queue time does not count toward timeout. If persistent, check service quotas and consider reserved capacity."

## Output Format

```yaml
root_cause: "Build timeout — <specific_cause>"
evidence:
  - type: build_phases
    content: "<phase durations>"
  - type: project_config
    content: "<timeout and cache settings>"
severity: MEDIUM
mitigation:
  immediate: "Increase timeout or optimize the slow phase"
  long_term: "Implement caching, optimize build scripts, use appropriate compute type"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If timeout is too low: increase timeoutInMinutes | GREEN |
| If a specific phase is slow: optimize that phase | GREEN |
| If build is stuck: check VPC/network configuration | GREEN |
| Enable caching (S3 or local) to speed up dependency downloads | GREEN |
| Use a larger compute type for CPU/memory-intensive builds | YELLOW |
| Optimize Docker builds: order layers, use multi-stage builds | GREEN |
| Split large builds into multiple smaller CodeBuild projects | YELLOW |
| Use parallel test execution | GREEN |
| Pre-bake dependencies into a custom build image | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Build timeout patterns may reveal build complexity and infrastructure constraints

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If timeout was increased, restore the previous timeout value: `aws codebuild update-project --name <project> --timeout-in-minutes <previous-value>`
2. If compute type was changed, restore the previous compute type
3. If caching was enabled, disable it if it causes issues
4. If the build was split into multiple projects, consolidate back if the split caused problems
5. Verify rollback by triggering a build and confirming expected behavior

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
