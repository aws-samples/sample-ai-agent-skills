---
title: "E2 — Circuit Breaker Rollback"
description: "Diagnose and respond to ECS deployment circuit breaker rollbacks"
status: active
severity: HIGH
triggers:
  - "Circuit breaker rollback"
  - "Deployment failed"
  - "SERVICE_DEPLOYMENT_FAILED"
owner: devops-agent
objective: "Understand why the circuit breaker triggered and fix the deployment"
context: "ECS deployment circuit breaker automatically rolls back a deployment after consecutive task failures. It monitors task launch failures and rolls back to the last successful deployment. Enabled via deploymentConfiguration.deploymentCircuitBreaker."
---

## Phase 1 — Triage

MUST:
- Check service events for circuit breaker rollback messages
- Check stopped tasks from the failed deployment for stoppedReason and exit codes
- Verify the task definition that was being deployed
- Check container logs from the failed tasks

SHOULD:
- Identify the specific failure: image pull, OOM, health check, crash
- Check if the rollback completed successfully
- Verify the service is now running the previous stable task definition

MAY:
- Check if the failure is environment-specific (works locally but not in ECS)
- Review the deployment configuration thresholds

## Common Issues

- symptoms: "Circuit breaker rolled back, tasks failing with exit code 1"
  diagnosis: "Application error in the new version. Code bug or missing configuration."
  resolution: "Fix the application error, test locally, then redeploy."

- symptoms: "Circuit breaker rolled back, image pull failures"
  diagnosis: "New image tag doesn't exist or ECR permissions changed."
  resolution: "Verify image exists in ECR, check execution role permissions."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
4. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
5. If health check settings were modified, restore original target group health check configuration
6. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "circuit_breaker_rollback — <underlying failure>"
evidence:
  - type: ecs_service
    content: "<circuit breaker event and stopped task details>"
severity: HIGH
mitigation:
  immediate: "Fix the underlying task failure and redeploy"
  long_term: "Test deployments in staging, use canary deployments"
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
