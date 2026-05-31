---
title: "D1 — Server Process Crashes"
description: "Diagnose GameLift server process crashes"
status: active
severity: HIGH
triggers: ["server crash", "process terminated", "game server down", "process exited", "server not responding"]
owner: devops-agent
objective: "Resolve game server process crashes and unexpected terminations"
context: "Server processes run on fleet instances. They must integrate GameLift Server SDK. Crashes trigger process restart up to the configured limit. Repeated crashes mark the instance unhealthy."
---

## Phase 1 — Triage

MUST:
- Check fleet events for process terminations: `aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>`
- Check instance status: `aws gamelift describe-instances --fleet-id <fleet-id> --query 'Instances[*].{Id:InstanceId,Status:Status}'`
- Request game session logs: `aws gamelift get-game-session-log-url --game-session-id <session-id>`

SHOULD:
- Check runtime configuration for correct launch path and parameters
- Verify server binary compatibility with the fleet OS (Amazon Linux 2)
- Check if crashes correlate with specific game session events

MAY:
- Enable remote access to inspect instance: `aws gamelift get-instance-access --fleet-id <fleet-id> --instance-id <instance-id>`
- Check CloudWatch metrics for process health: `aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name ServerProcessAbnormalTerminations --dimensions Name=FleetId,Value=<fleet-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

## Phase 2 — Remediate

MUST:
- Fix server binary crashes: check for segfaults, out-of-memory, or unhandled exceptions
- Ensure all shared libraries are included in the build
- Verify GameLift Server SDK is properly initialized

SHOULD:
- Add crash logging to the server process for better diagnostics
- Configure appropriate server process count per instance based on resource needs
- Implement graceful error handling in the game server

MAY:
- Use core dumps for crash analysis (if remote access is enabled)
- Implement watchdog logging to track process health

## Safety Ratings

```
safety_ratings:
  - "describe-fleet-events / describe-instances: GREEN — read-only inspection"
  - "get-game-session-log-url: GREEN — read-only log access"
  - "get-instance-access: YELLOW — provides SSH credentials to fleet instance"
  - "update-runtime-configuration: YELLOW — changes server process configuration"
```

## Escalation Conditions

- All server processes on all instances crashing (build-level issue)
- Crashes only occur under specific load conditions (resource exhaustion)
- Crashes correlate with specific player actions (game logic bug)
- Instance access needed but remote access was not enabled during fleet creation

## Data Sensitivity

- `aws gamelift describe-fleet-events`: LOW — fleet lifecycle events
- `aws gamelift get-game-session-log-url`: MEDIUM — game session logs may contain player data
- `aws gamelift get-instance-access`: HIGH — provides SSH credentials
- Core dumps: HIGH — may contain memory contents including player data

## Prohibited Actions

- NEVER share instance SSH credentials externally
- NEVER modify server binaries on running instances (changes don't persist)
- NEVER ignore repeated crashes — they indicate a systemic issue
- NEVER disable health checks to mask crash issues

## Phase 3 — Rollback

1. If runtime configuration change caused crashes: restore original configuration
2. If new build caused crashes: create new fleet with previous build version
3. If server process count was changed: restore original count
4. Scale up healthy instances to maintain capacity while investigating
5. Document crash patterns and update server testing procedures

## Common Issues

- symptoms: "Server processes crash immediately after starting with 'shared library not found'"
  diagnosis: "Build is missing required shared libraries (.so files) that the server binary depends on."
  resolution: "Include all shared libraries in the build. Use ldd <binary> locally to identify dependencies. Bundle them in the build directory."

## Output Format

```yaml
root_cause: "server_process_crash — <specific_cause>"
evidence:
  - type: crash_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Stabilize server processes"
  long_term: "Implement comprehensive crash logging and pre-deployment testing"
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
