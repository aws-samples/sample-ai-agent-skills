---
title: "D2 — Health Check Failures"
description: "Diagnose GameLift server process health check failures"
status: active
severity: HIGH
triggers: ["health check failed", "unhealthy process", "instance unhealthy", "ProcessReady not called"]
owner: devops-agent
objective: "Resolve server process health check and SDK integration issues"
context: "GameLift Server SDK health checks verify server processes are responsive. Processes must call ProcessReady() and respond to onHealthCheck(). Failed health checks trigger process termination."
---

## Phase 1 — Triage

MUST:
- Check fleet events for health check failures: `aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>`
- Verify server process is calling ProcessReady(): check server logs
- Check instance health: `aws gamelift describe-instances --fleet-id <fleet-id>`

SHOULD:
- Verify onHealthCheck() callback returns true within 60 seconds
- Check if health check failures correlate with high CPU/memory usage
- Review runtime configuration for correct launch path

MAY:
- Get instance access for direct debugging: `aws gamelift get-instance-access --fleet-id <fleet-id> --instance-id <instance-id>`
- Check CloudWatch for health metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name HealthyServerProcesses --dimensions Name=FleetId,Value=<fleet-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

## Phase 2 — Remediate

MUST:
- Ensure server calls ProcessReady() with correct port and log paths on startup
- Implement onHealthCheck() callback that returns true when server is healthy
- Ensure health check response completes within 60-second timeout

SHOULD:
- Add logging around SDK calls for debugging
- Ensure server process does not block the main thread during health checks
- Monitor resource usage to prevent health check timeouts due to resource exhaustion

MAY:
- Implement detailed health status reporting in onHealthCheck()
- Use Anywhere fleets for development to test SDK integration locally

## Safety Ratings

```
safety_ratings:
  - "describe-fleet-events / describe-instances: GREEN — read-only inspection"
  - "get-instance-access: YELLOW — provides SSH credentials"
  - "update-runtime-configuration: YELLOW — changes process configuration"
  - "stop-fleet-actions (auto-scaling): YELLOW — pauses scaling during investigation"
```

## Escalation Conditions

- All processes failing health checks across all instances
- Health check failures only under load (resource exhaustion)
- SDK integration issues requiring game server code changes
- Fleet stuck in ACTIVATING because no process reports healthy

## Data Sensitivity

- `aws gamelift describe-fleet-events`: LOW — fleet events
- `aws gamelift get-instance-access`: HIGH — SSH credentials
- Server process logs: MEDIUM — may contain game state data
- Health check responses: LOW — boolean health status

## Prohibited Actions

- NEVER disable health checks (they are mandatory for GameLift)
- NEVER implement onHealthCheck() to always return true without actual checks
- NEVER ignore health check failures — they indicate real server issues
- NEVER modify SDK integration without thorough testing

## Phase 3 — Rollback

1. If SDK integration change caused failures: revert server code and deploy new build
2. If runtime configuration change caused issues: restore original configuration
3. If resource limits caused health check timeouts: reduce server processes per instance
4. Scale up instances to maintain capacity while fixing health check issues
5. Document health check patterns and update server development guidelines

## Common Issues

- symptoms: "Fleet stuck in ACTIVATING — no server processes report healthy"
  diagnosis: "Server process is not calling GameLift Server SDK ProcessReady() on startup."
  resolution: "Verify server integrates GameLift Server SDK. Call ProcessReady() with port number and log paths after server initialization completes."

## Output Format

```yaml
root_cause: "health_check_failure — <specific_cause>"
evidence:
  - type: health_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix health check integration"
  long_term: "Implement comprehensive SDK integration testing in CI/CD"
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
