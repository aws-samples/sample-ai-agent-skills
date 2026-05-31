---
title: "B3 — Image Not Found"
description: "Diagnose image pull failures caused by missing or incorrect image references"
status: active
severity: HIGH
triggers:
  - "CannotPullContainerError"
  - "manifest unknown"
  - "image not found"
  - "repository does not exist"
owner: devops-agent
objective: "Fix image reference and restore task deployment"
context: "Image reference format: account.dkr.ecr.region.amazonaws.com/repo:tag. Common issues: wrong tag, wrong region, deleted image, typo in repository name."
---

## Phase 1 — Triage

MUST:
- Check the image reference in the task definition
- Verify the ECR repository exists: `aws ecr describe-repositories --repository-names <repo>`
- Verify the image tag exists: `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>`
- Check if the image was recently deleted (lifecycle policy or manual)

SHOULD:
- Verify the region in the image URI matches the task's region
- Check if the tag was overwritten with a broken image
- Verify the image architecture matches (x86_64 vs arm64)

MAY:
- Check ECR lifecycle policies for automatic image cleanup
- Verify if using 'latest' tag and it was recently updated

## Common Issues

- symptoms: "manifest unknown, tag doesn't exist"
  diagnosis: "Image tag was never pushed or was deleted by lifecycle policy."
  resolution: "Push the image with the correct tag, or update task definition to use an existing tag."

- symptoms: "Repository does not exist"
  diagnosis: "Typo in repository name or repository in different region/account."
  resolution: "Fix the repository name/region in the task definition image URI."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- Service handles production traffic or is business-critical
- Fix requires modifying the task definition for a running service
- Resolution involves changing capacity provider or cluster configuration
- Multiple services affected suggesting cluster-level or account-level issue
- Deployment rollback has been triggered by the circuit breaker

## Data Sensitivity

HIGH - ECS task definitions may contain secrets ARNs, environment variables with credentials, and sensitive configuration. Container logs may capture application data including PII. Service Connect and service discovery configurations may expose internal architecture.

## Prohibited Actions

- NEVER suggest stopping all tasks in a service to fix a deployment issue - this causes a full outage
- NEVER suggest removing or modifying the task execution role while tasks are running
- NEVER recommend deleting a service without first scaling desired count to 0 and draining connections
- NEVER suggest modifying a running task definition - create a new revision instead
- NEVER recommend disabling the deployment circuit breaker on a production service

## Phase 3 - Rollback

1. If task definition was updated, roll back to previous revision: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <previous-revision>`
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
3. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
## Output Format

```yaml
root_cause: "image_not_found — <detail>"
evidence:
  - type: ecr
    content: "<image reference and repository state>"
severity: HIGH
mitigation:
  immediate: "Fix image reference or push the missing image"
  long_term: "Use immutable tags, protect critical images from lifecycle policies"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
