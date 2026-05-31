---
title: "B3 — Build Out of Memory (OOM)"
description: "Diagnose and resolve CodeBuild out-of-memory failures"
status: active
severity: HIGH
triggers:
  - "out of memory"
  - "OOM"
  - "memory exhausted"
  - "Cannot allocate memory"
  - "ENOMEM"
  - "Java heap space"
owner: devops-agent
objective: "Identify why the build ran out of memory and resolve the issue"
context: "CodeBuild compute types have fixed memory limits: BUILD_GENERAL1_SMALL (3GB), BUILD_GENERAL1_MEDIUM (7GB), BUILD_GENERAL1_LARGE (15GB), BUILD_GENERAL1_2XLARGE (145GB). OOM occurs when build processes exceed the available memory. Common causes include large compilation jobs, memory-hungry test suites, Docker builds, and memory leaks in build tools."
---

## Phase 1 — Triage

MUST:
- Get build details and check for OOM indicators: `aws codebuild batch-get-builds --ids <build-id>`
- Check the compute type: `aws codebuild batch-get-projects --names <project-name> --query 'projects[0].environment.computeType'`
- Check build logs for memory-related errors: `aws logs get-log-events --log-group-name /aws/codebuild/<project-name> --log-stream-name <build-id>`
- Identify which phase failed with OOM

SHOULD:
- Check if the build runs Docker (Docker daemon consumes additional memory)
- Review build commands for memory-intensive operations (parallel compilation, large test suites)
- Check if Java/Node.js heap size is configured appropriately

MAY:
- Compare memory usage patterns across recent builds
- Check if the build image itself is memory-heavy
- Review Dockerfile for multi-stage build opportunities

## Phase 2 — Remediate

MUST:
- If compute type is too small: upgrade to a larger compute type
- If Java heap: set appropriate `-Xmx` flag (e.g., `-Xmx4g` for MEDIUM)
- If Node.js heap: set `--max-old-space-size` (e.g., `NODE_OPTIONS=--max-old-space-size=4096`)
- If Docker build OOM: limit Docker build memory or use a larger compute type

SHOULD:
- Reduce parallelism in build commands (e.g., `make -j2` instead of `make -j$(nproc)`)
- Use multi-stage Docker builds to reduce memory during image construction
- Split memory-intensive tests into separate build projects

MAY:
- Use BUILD_GENERAL1_2XLARGE for extremely memory-intensive builds
- Implement build-time memory monitoring with `free -m` commands in buildspec
- Consider ARM-based compute types (BUILD_GENERAL1_SMALL with ARM) for better memory efficiency

## Common Issues

- symptoms: "Build killed — Cannot allocate memory"
  diagnosis: "Build process exceeded the compute type's memory limit."
  resolution: "Upgrade compute type: `aws codebuild update-project --name <project> --environment computeType=BUILD_GENERAL1_LARGE`."

- symptoms: "java.lang.OutOfMemoryError: Java heap space"
  diagnosis: "Java process heap size exceeds available memory."
  resolution: "Set JAVA_OPTS or MAVEN_OPTS: `export MAVEN_OPTS='-Xmx4g'` in buildspec."

- symptoms: "JavaScript heap out of memory during npm build"
  diagnosis: "Node.js process exceeded default heap limit."
  resolution: "Set NODE_OPTIONS: `export NODE_OPTIONS=--max-old-space-size=4096` in buildspec."

- symptoms: "Docker build killed during layer creation"
  diagnosis: "Docker daemon plus build process exceed available memory."
  resolution: "Use larger compute type. Docker daemon itself uses ~500MB. Use multi-stage builds to reduce peak memory."

## Output Format

```yaml
root_cause: "Build OOM — <specific_cause>"
evidence:
  - type: build_logs
    content: "<OOM error message>"
  - type: project_config
    content: "<compute type and memory limit>"
severity: HIGH
mitigation:
  immediate: "Upgrade compute type or reduce memory usage"
  long_term: "Optimize build for memory efficiency, set appropriate heap limits"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If compute type is too small: upgrade to a larger compute type | YELLOW |
| If Java heap: set appropriate -Xmx flag | GREEN |
| If Node.js heap: set --max-old-space-size | GREEN |
| If Docker build OOM: limit Docker build memory or use larger compute type | YELLOW |
| Reduce parallelism in build commands | GREEN |
| Use multi-stage Docker builds | GREEN |
| Split memory-intensive tests into separate build projects | YELLOW |
| Use BUILD_GENERAL1_2XLARGE for extremely memory-intensive builds | YELLOW |
| Implement build-time memory monitoring | GREEN |
| Consider ARM-based compute types | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- OOM errors in build logs may expose memory-sensitive application data

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If compute type was upgraded, restore the previous compute type: `aws codebuild update-project --name <project> --environment computeType=<previous-type>`
2. If heap size settings were changed, restore previous JAVA_OPTS or NODE_OPTIONS values
3. If build parallelism was reduced, restore previous parallelism settings
4. If builds were split into multiple projects, consolidate back if needed
5. Verify rollback by triggering a build and confirming it completes without OOM

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
