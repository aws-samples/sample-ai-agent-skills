---
title: "B3 — App Runner Port Configuration Issues"
description: "Diagnose port mismatch issues between App Runner and the application"
status: active
severity: HIGH
triggers:
  - "port mismatch"
  - "wrong port"
  - "PORT env"
  - "connection refused"
  - "port 8080"
  - "listen port"
owner: devops-agent
objective: "Identify and fix port configuration mismatches between App Runner and the application"
context: "App Runner sets the PORT environment variable and routes all traffic (including health checks) to that port. The application must listen on the PORT env var value. If the application hardcodes a different port, App Runner cannot reach it, causing health check failures and service unavailability. The default port is 8080."
---

## Phase 1 — Triage

MUST:
- Check the configured port: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.InstanceConfiguration.Port'`
- Review application code or Dockerfile for hardcoded port values
- Check application startup logs for the port it's listening on: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`
- Verify the application reads the PORT environment variable
- Check health check configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.HealthCheckConfiguration'`

SHOULD:
- Check if the Dockerfile EXPOSE directive matches the configured port (informational only — EXPOSE does not affect App Runner routing)
- Verify the start command does not override the port with a hardcoded value
- Check if a framework-specific configuration file overrides the port

MAY:
- Test locally with PORT=8080 to verify the application respects the env var
- Check if the application binds to 0.0.0.0 (required) vs 127.0.0.1 (will fail)

## Phase 2 — Remediate

MUST:
- Update the application to read the PORT environment variable at startup
- Ensure the application binds to 0.0.0.0 (all interfaces), not 127.0.0.1 or localhost
- If the application cannot be changed, set the App Runner port to match the application's hardcoded port

SHOULD:
- Use framework-standard port configuration: Node.js `process.env.PORT`, Python `os.environ.get('PORT', '8080')`, Go `os.Getenv("PORT")`, Java `System.getenv("PORT")`
- Document the port requirement in the application README

MAY:
- Add a startup log line confirming the port the application is listening on
- Use a wrapper script that sets the port before starting the application

## Common Issues

- symptoms: "Health checks fail but application works locally on port 3000"
  diagnosis: "Application hardcodes port 3000 but App Runner sends traffic to port 8080 (default)."
  resolution: "Change app to use process.env.PORT || 3000, or set App Runner port to 3000."

- symptoms: "Application starts but returns connection refused"
  diagnosis: "Application binds to 127.0.0.1 instead of 0.0.0.0."
  resolution: "Change the application to bind to 0.0.0.0 or use the host '0.0.0.0' in the listen call."

- symptoms: "Dockerfile EXPOSE 3000 but App Runner uses 8080"
  diagnosis: "EXPOSE is documentation only — it does not configure App Runner's port routing."
  resolution: "Set the port in App Runner service configuration or make the app read PORT env var."

## Output Format

```yaml
root_cause: "port_configuration — <specific_cause>"
evidence:
  - type: configured_port
    content: "<App Runner port setting>"
  - type: application_port
    content: "<port the application actually listens on>"
  - type: startup_logs
    content: "<log showing port binding>"
severity: HIGH
mitigation:
  immediate: "Align application port with App Runner PORT configuration"
  long_term: "Always use PORT env var in application code for portability"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update app to read PORT env var | YELLOW | Application code change |
| Ensure app binds to 0.0.0.0 | YELLOW | Application code change |
| Set App Runner port to match app | YELLOW | Service configuration change |
| Add startup log for port confirmation | GREEN | Logging improvement |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Port configuration**: Application and service port settings — INTERNAL
- **Application code**: Port binding logic — INTERNAL
- **Startup logs**: Port binding confirmation — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER change the port configuration without verifying the application supports it

## Phase 3 — Rollback

1. Revert application code to previous port binding logic
2. Revert App Runner port configuration to previous value
3. Redeploy with the reverted configuration
4. Verify health checks pass on the correct port
5. Confirm application is accessible via the App Runner URL

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
