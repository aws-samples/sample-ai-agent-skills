---
title: "D1 — Local Development Issues"
description: "Diagnose local development and testing issues with SAM CLI"
status: active
severity: MEDIUM
triggers: ["sam local error", "local invoke failed", "Docker error", "local development not working"]
owner: devops-agent
objective: "Resolve local development and testing issues for Application Composer projects"
context: "Local development with SAM CLI requires Docker for Lambda emulation. Local testing does not perfectly replicate production."
---

## Phase 1 — Triage

MUST:
- Verify Docker is running: `docker info 2>&1 | head -5`
- Check SAM CLI version: `sam --version`
- Attempt a local invoke: `sam local invoke <FunctionName> --event event.json --debug 2>&1 | tail -30`

SHOULD:
- Verify Docker socket permissions: `ls -la /var/run/docker.sock`
- Check available disk space for Docker images: `docker system df`

MAY:
- Review Docker container logs: `docker logs $(docker ps -lq)`
- Check for port conflicts: `lsof -i :3000`

## Phase 2 — Remediate

MUST:
- Start Docker if not running
- Fix Docker socket permissions if access denied
- Ensure SAM build completes before local invoke: `sam build`

SHOULD:
- Pull the correct Lambda runtime image: `docker pull public.ecr.aws/sam/build-<runtime>`
- Set environment variables for local testing: `sam local invoke --env-vars env.json`

MAY:
- Use sam local start-api for API Gateway emulation
- Configure local DynamoDB for integration testing

## Safety Ratings

```
safety_ratings:
  - "docker info: GREEN — read-only Docker status check"
  - "sam local invoke: GREEN — local execution only, no AWS resources affected"
  - "sam build: YELLOW — creates build artifacts locally"
  - "docker pull: YELLOW — downloads container images"
```

## Escalation Conditions

- Docker consistently fails to start Lambda runtime containers
- Local invoke produces different results than deployed Lambda
- SAM build fails with dependency resolution errors
- Docker images consume excessive disk space

## Data Sensitivity

- `docker info`: LOW — reveals Docker engine configuration
- `sam local invoke`: LOW — local execution with test events
- `docker logs`: MEDIUM — may contain function output including sensitive data from test events

## Prohibited Actions

- NEVER use production AWS credentials for local testing without understanding the implications
- NEVER assume local test results match production behavior exactly
- NEVER run sam local invoke with production event data containing PII
- NEVER disable Docker security features to fix permission issues

## Phase 3 — Rollback

1. Clean build artifacts if corrupted: `sam build --clean`
2. Remove problematic Docker images: `docker rmi <image-id>`
3. Reset Docker if containers are stuck: `docker system prune -f`
4. Restore event.json and env.json from version control
5. Verify local invoke works with a minimal test event

## Common Issues

- symptoms: "sam local invoke fails with 'Error: Running AWS SAM projects locally requires Docker'"
  diagnosis: "Docker is not installed, not running, or the user lacks Docker socket permissions."
  resolution: "Install Docker, start Docker Desktop, or add user to docker group: sudo usermod -aG docker $USER"

## Output Format

```yaml
root_cause: "local_development — <specific_cause>"
evidence:
  - type: local_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Docker/SAM CLI setup"
  long_term: "Document local development prerequisites"
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
