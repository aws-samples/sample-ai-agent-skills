---
title: "B4 — Docker Build Issues"
description: "Diagnose and resolve Docker build failures in CodeBuild"
status: active
severity: HIGH
triggers:
  - "Docker build failed"
  - "docker daemon"
  - "privileged mode"
  - "ECR login"
  - "image pull"
  - "docker push"
owner: devops-agent
objective: "Identify why Docker builds are failing in CodeBuild and resolve the issue"
context: "Docker builds in CodeBuild require privileged mode enabled. Common issues include Docker daemon not starting, ECR authentication failures, image pull/push permission errors, Docker layer cache misses, and Dockerfile syntax errors. CodeBuild supports Docker-in-Docker via privileged mode."
---

## Phase 1 — Triage

MUST:
- Check if privileged mode is enabled: `aws codebuild batch-get-projects --names <project-name> --query 'projects[0].environment.privilegedMode'`
- Check build logs for Docker errors: `aws logs get-log-events --log-group-name /aws/codebuild/<project-name> --log-stream-name <build-id>`
- Verify the build image supports Docker: check `environment.image` in project config
- Check if ECR login is in the buildspec pre_build phase

SHOULD:
- Verify CodeBuild service role has ECR permissions: `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`
- Check if Docker layer caching is configured: `aws codebuild batch-get-projects --names <project-name> --query 'projects[0].cache'`
- Verify ECR repository exists: `aws ecr describe-repositories --repository-names <repo>`

MAY:
- Check if the build is in a VPC without NAT (cannot pull base images)
- Review Dockerfile for syntax errors or missing base images
- Check ECR lifecycle policies that may have deleted required images

## Phase 2 — Remediate

MUST:
- If privileged mode disabled: enable it: `aws codebuild update-project --name <project> --environment privilegedMode=true`
- If ECR login missing: add `aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com` to pre_build
- If permission denied on push: update CodeBuild service role with ECR write permissions
- If base image pull fails: verify image exists and is accessible (public or authenticated)

SHOULD:
- Enable Docker layer caching for faster builds (requires privileged mode)
- Use ECR for base images instead of Docker Hub (avoids rate limiting)
- Use multi-stage Dockerfile to reduce final image size

MAY:
- Use `docker buildx` for multi-platform builds
- Implement image scanning in the build pipeline: `aws ecr start-image-scan`
- Cache Docker layers in ECR as a build cache source

## Common Issues

- symptoms: "Cannot connect to the Docker daemon"
  diagnosis: "Privileged mode is not enabled. Docker daemon cannot start without it."
  resolution: "Enable privileged mode in the CodeBuild project environment settings."

- symptoms: "Error response from daemon: pull access denied"
  diagnosis: "Docker Hub rate limiting or missing ECR authentication."
  resolution: "For ECR: add ecr get-login-password in pre_build. For Docker Hub: use authenticated pulls or switch to ECR."

- symptoms: "denied: Your authorization token has expired"
  diagnosis: "ECR authorization token expired. Tokens are valid for 12 hours."
  resolution: "Ensure ECR login is in pre_build phase (runs fresh each build). Don't cache ECR tokens."

- symptoms: "Docker build slow — no layer cache"
  diagnosis: "Docker layer cache not configured or cache miss due to different build host."
  resolution: "Enable DOCKER_LAYER_CACHE in project cache settings. Consider using ECR as a remote cache."

## Output Format

```yaml
root_cause: "Docker build issue — <specific_cause>"
evidence:
  - type: build_logs
    content: "<Docker error message>"
  - type: project_config
    content: "<privileged mode, cache, environment settings>"
severity: HIGH
mitigation:
  immediate: "Fix Docker configuration and retry build"
  long_term: "Use ECR for base images, enable layer caching, implement multi-stage builds"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If privileged mode disabled: enable it | YELLOW |
| If ECR login missing: add ecr get-login-password to pre_build | GREEN |
| If permission denied on push: update CodeBuild service role with ECR write permissions | YELLOW |
| If base image pull fails: verify image exists and is accessible | GREEN |
| Enable Docker layer caching for faster builds | GREEN |
| Use ECR for base images instead of Docker Hub | GREEN |
| Use multi-stage Dockerfile to reduce final image size | GREEN |
| Use docker buildx for multi-platform builds | GREEN |
| Implement image scanning: aws ecr start-image-scan | GREEN |
| Cache Docker layers in ECR | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Docker build logs may contain base image credentials, build arguments, and application secrets

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If privileged mode was enabled, disable it if no longer needed: `aws codebuild update-project --name <project> --environment privilegedMode=false`
2. If ECR permissions were broadened, revert the CodeBuild service role policy
3. If Docker layer caching was enabled, disable it if it causes issues
4. If base images were switched to ECR, keep the ECR images (reverting may break builds)
5. Verify rollback by triggering a Docker build and confirming it completes successfully

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
