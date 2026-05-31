---
title: "B1 — App Runner Health Check Failures"
description: "Diagnose health check failures causing deployment rollbacks or unhealthy services"
status: active
severity: HIGH
triggers:
  - "health check failed"
  - "health check timeout"
  - "unhealthy"
  - "deployment rollback"
  - "service not healthy"
owner: devops-agent
objective: "Identify why App Runner health checks are failing and restore service health"
context: "App Runner performs health checks to verify the application is running correctly. Failed health checks during deployment cause automatic rollback. App Runner supports TCP health checks (port open) and HTTP health checks (200 response on a path). The most common cause is the application not listening on the PORT environment variable."
---

## Phase 1 — Triage

MUST:
- Check health check configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.HealthCheckConfiguration'`
- Check the configured port: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.InstanceConfiguration.Port'`
- Review application logs for startup errors: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`
- Check service status and recent operations: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.{Status:Status,HealthCheck:HealthCheckConfiguration}'`
- Verify the application reads and listens on the PORT environment variable

SHOULD:
- Check if the health check path exists and returns HTTP 200 (for HTTP health checks)
- Verify the application starts within the health check window (interval × unhealthy threshold)
- Check if the application has slow startup that exceeds the health check timeout

MAY:
- Test the health check endpoint locally
- Check if the application requires environment variables or secrets to start successfully

## Phase 2 — Remediate

MUST:
- Ensure the application listens on the port specified by the PORT environment variable (default 8080)
- For HTTP health checks, ensure the health check path returns HTTP 200 when the application is ready
- Increase unhealthy threshold or interval if the application has a slow startup

SHOULD:
- Use TCP health checks for simpler validation (just checks port is open)
- Implement a dedicated /health endpoint that checks application readiness
- Set appropriate timeout values — default 2 seconds may be too short for complex health checks

MAY:
- Add startup logging to confirm the application is listening on the correct port
- Implement graceful startup that responds to health checks before full initialization

## Common Issues

- symptoms: "Deployment rolls back after a few minutes with health check failure"
  diagnosis: "Application hardcodes port 3000 but App Runner sends health checks to port 8080 (default PORT)."
  resolution: "Update application to read PORT env var: const port = process.env.PORT || 8080."

- symptoms: "HTTP health check returns 404"
  diagnosis: "Health check path does not exist in the application."
  resolution: "Add the health check endpoint (e.g., /health) or change the health check path to an existing route."

- symptoms: "Health check times out during startup"
  diagnosis: "Application takes longer to start than the health check window allows."
  resolution: "Increase interval (e.g., 10s) and unhealthy threshold (e.g., 10) to allow more startup time."

## Output Format

```yaml
root_cause: "health_check_failure — <specific_cause>"
evidence:
  - type: health_check_config
    content: "<protocol, path, interval, timeout, thresholds>"
  - type: port_config
    content: "<configured port and PORT env var>"
  - type: application_logs
    content: "<startup logs showing port binding>"
severity: HIGH
mitigation:
  immediate: "Fix port configuration or health check settings and redeploy"
  long_term: "Implement proper health check endpoint and startup readiness checks"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure app listens on PORT env var | YELLOW | Application code change |
| Ensure health check path returns 200 | YELLOW | Application code change |
| Increase unhealthy threshold/interval | GREEN | Health check tuning |
| Use TCP health checks | GREEN | Health check type change |
| Implement /health endpoint | YELLOW | Application code change |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Health check config**: Protocol, path, thresholds — INTERNAL
- **Port configuration**: Application listening port — INTERNAL
- **Application logs**: Startup and health check responses — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER disable health checks to work around application issues

## Phase 3 — Rollback

1. App Runner automatically rolls back on health check failure during deployment
2. Revert health check configuration if it was modified: update-service with previous settings
3. Revert application code changes and redeploy with previous version
4. Verify service returns to healthy state
5. Monitor health check metrics after rollback

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
