---
title: "E1 — Timeout and Disconnection"
description: "Diagnose CloudShell session timeout and disconnection issues"
status: active
severity: MEDIUM
triggers: ["session timed out", "disconnected", "lost connection", "session expired", "reconnect failed"]
owner: devops-agent
objective: "Resolve CloudShell session timeout and disconnection issues"
context: "Sessions time out after 20 minutes of inactivity. Maximum session duration is 12 hours. WebSocket connections can be interrupted by network issues."
---

## Phase 1 — Triage

MUST:
- Determine if timeout was due to inactivity (20 min) or max duration (12 hours)
- Check if running processes were lost: review command history after reconnection
- Verify network stability: check for corporate proxy or VPN interruptions

SHOULD:
- Check if screen/tmux was in use to preserve session state
- Review browser console for WebSocket disconnection errors (F12 → Console)

MAY:
- Check CloudTrail for StopEnvironment events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StopEnvironment --max-results 10`
- Monitor session patterns to identify recurring disconnection times

## Phase 2 — Remediate

MUST:
- Use screen or tmux for long-running processes: `screen -S mysession` or `tmux new -s mysession`
- Reconnect to existing screen/tmux sessions after timeout: `screen -r mysession` or `tmux attach -t mysession`
- Save work frequently to $HOME or S3

SHOULD:
- Add keepalive mechanism for critical sessions (periodic keyboard input)
- Use nohup for background processes: `nohup ./long-script.sh > $HOME/output.log 2>&1 &`

MAY:
- Configure browser to prevent sleep/suspension during CloudShell sessions
- Use CloudShell API to monitor environment status programmatically

## Safety Ratings

```
safety_ratings:
  - "screen / tmux: GREEN — session multiplexer, no AWS resource changes"
  - "nohup: GREEN — runs process in background, no AWS resource changes"
  - "StopEnvironment: YELLOW — stops the environment, preserves storage"
  - "DeleteEnvironment: RED — permanently removes environment and storage"
```

## Escalation Conditions

- Sessions disconnect immediately after starting (possible service issue)
- WebSocket connections blocked by corporate firewall
- Timeout occurs during critical operations that cannot be interrupted
- Multiple users experiencing simultaneous disconnections

## Data Sensitivity

- `screen / tmux session list`: LOW — shows session names only
- `aws cloudtrail lookup-events`: MEDIUM — reveals session activity
- Browser console logs: LOW — WebSocket connection metadata
- Command history: MEDIUM — may contain sensitive commands

## Prohibited Actions

- NEVER run production-critical operations in CloudShell without session protection (screen/tmux)
- NEVER rely on CloudShell for unattended long-running processes
- NEVER store session recordings containing sensitive data
- NEVER attempt to bypass the 12-hour maximum session limit

## Phase 3 — Rollback

1. If a process was interrupted mid-execution: check process state and resume or restart
2. If screen/tmux session was lost: check if process completed by reviewing output files
3. If data was being transferred when disconnected: verify transfer completeness with checksums
4. If CloudFormation stack operation was interrupted: check stack status and wait for completion
5. Document the interrupted operation and verify system state

## Common Issues

- symptoms: "CloudShell disconnects after exactly 20 minutes every time"
  diagnosis: "Session inactivity timeout. No keyboard or mouse input detected for 20 minutes."
  resolution: "Use screen or tmux to preserve session state. For automated tasks, use nohup or redirect output to files."

## Output Format

```yaml
root_cause: "timeout_disconnection — <specific_cause>"
evidence:
  - type: timeout_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Reconnect and recover session state"
  long_term: "Use screen/tmux for all long-running operations"
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
