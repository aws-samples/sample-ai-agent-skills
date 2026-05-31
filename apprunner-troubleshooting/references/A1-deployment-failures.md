---
title: "A1 — App Runner Deployment Failures"
description: "Diagnose why an App Runner service deployment failed or is stuck"
status: active
severity: HIGH
triggers:
  - "deployment failed"
  - "deployment stuck"
  - "CREATE_FAILED"
  - "UPDATE_FAILED"
  - "operation failed"
  - "deployment rollback"
owner: devops-agent
objective: "Identify why an App Runner deployment failed and restore the service to a healthy state"
context: "App Runner deployments can fail during service creation, updates, or redeployments. Failures trigger automatic rollback to the previous working version. Common causes include health check failures, invalid configuration, image pull errors, build failures, and IAM permission issues."
---

## Phase 1 — Triage

MUST:
- Check service status: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.Status'`
- List recent operations: `aws apprunner list-operations --service-arn <service-arn> --max-results 5`
- Check operation details for error messages in the operation status
- Review CloudWatch Logs for the service: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`
- Check CloudTrail for API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=apprunner.amazonaws.com --max-results 10`

SHOULD:
- Verify the source configuration is valid: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration'`
- Check health check configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.HealthCheckConfiguration'`
- Verify instance role exists and has correct trust policy: `aws iam get-role --role-name <role-name>`

MAY:
- Check if there are concurrent operations blocking the deployment
- Review service event logs in the console for detailed error messages

## Phase 2 — Remediate

MUST:
- If health check failure: verify the application starts and responds on the configured PORT (see B1)
- If image pull error: verify ECR image exists and access role has permissions (see A3)
- If build failure: check build logs for compilation or dependency errors (see A2)
- If IAM error: verify instance role trust policy includes apprunner.amazonaws.com

SHOULD:
- Wait for any in-progress operations to complete before retrying
- Use `aws apprunner start-deployment --service-arn <service-arn>` to trigger a manual redeployment after fixing the issue
- Check if the service is in DELETE_FAILED state which requires support intervention

MAY:
- Create a new service if the existing one is in an unrecoverable state
- Enable automatic rollback monitoring with CloudWatch alarms

## Common Issues

- symptoms: "Deployment fails immediately after starting"
  diagnosis: "Invalid source configuration — ECR image does not exist or access role lacks permissions."
  resolution: "Verify ECR image URI and tag. Ensure the access role has ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:GetDownloadUrlForLayer."

- symptoms: "Deployment starts but rolls back after a few minutes"
  diagnosis: "Health check failures — application does not respond on the configured port/path."
  resolution: "Verify application listens on PORT env var. Check health check path returns HTTP 200."

- symptoms: "Service stuck in OPERATION_IN_PROGRESS"
  diagnosis: "A previous operation has not completed or failed silently."
  resolution: "Wait for the operation to complete (up to 20 minutes). If stuck, contact AWS Support."

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: operation_status
    content: "<operation details and error message>"
  - type: service_status
    content: "<current service status>"
  - type: logs
    content: "<relevant log entries>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and trigger a new deployment"
  long_term: "Implement pre-deployment validation and health check testing"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify health check configuration | GREEN | Read-only check |
| Verify ECR image and access role | GREEN | Read-only check |
| Check build logs | GREEN | Read-only check |
| Verify instance role trust policy | GREEN | Read-only check |
| Trigger manual redeployment | RED | Deploys new version to production |
| Create a new service | RED | New production service creation |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Service configuration**: Source, health check, instance settings — INTERNAL
- **CloudWatch Logs**: Application output and errors — INTERNAL
- **CloudTrail events**: API call history — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER delete a production service to recreate it without stakeholder approval

## Phase 3 — Rollback

1. App Runner automatically rolls back to the previous working version on deployment failure
2. If manual redeployment was triggered, wait for automatic rollback to complete
3. Verify service status returns to RUNNING with previous version
4. Check application logs to confirm previous version is serving traffic
5. Investigate root cause before attempting another deployment

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
