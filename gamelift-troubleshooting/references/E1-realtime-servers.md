---
title: "E1 — Realtime Servers"
description: "Diagnose GameLift Realtime Servers issues"
status: active
severity: MEDIUM
triggers: ["Realtime Server", "Realtime script", "JavaScript server", "lightweight server"]
owner: devops-agent
objective: "Resolve Realtime Servers configuration and runtime issues"
context: "Realtime Servers use JavaScript scripts for lightweight game servers. They provide built-in messaging and game session management. Different from custom game servers using compiled builds."
---

## Phase 1 — Triage

MUST:
- Check script status: `aws gamelift describe-script --script-id <script-id>`
- Check fleet using the script: `aws gamelift describe-fleet-attributes --fleet-ids <fleet-id>`
- Review fleet events for script errors: `aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>`

SHOULD:
- Verify script implements required callbacks (onStartGameSession, onProcessStarted)
- Check game session logs for JavaScript runtime errors
- Verify client SDK version is compatible with Realtime Server version

MAY:
- Test script logic locally with Node.js
- Check CloudWatch metrics for Realtime Server performance

## Phase 2 — Remediate

MUST:
- Fix JavaScript syntax errors in the script
- Implement required callback functions: onStartGameSession, onProcessStarted, onMessage
- Update script: `aws gamelift update-script --script-id <script-id> --zip-file fileb://fixed-script.zip`

SHOULD:
- Add error handling and logging to script callbacks
- Test message handling with sample client connections
- Verify script handles player connect/disconnect events

MAY:
- Implement custom game logic in script callbacks
- Use Realtime Server built-in messaging for common patterns

## Safety Ratings

```
safety_ratings:
  - "describe-script / describe-fleet-attributes: GREEN — read-only inspection"
  - "update-script: YELLOW — updates script on running fleets (rolling update)"
  - "create-fleet (Realtime): YELLOW — creates instances with script"
  - "delete-script: RED — removes script, may affect fleets"
```

## Escalation Conditions

- Script update causes fleet-wide game session failures
- JavaScript runtime errors not reproducible locally
- Client SDK incompatibility with Realtime Server version
- Performance issues under load requiring architecture changes

## Data Sensitivity

- `aws gamelift describe-script`: LOW — script metadata
- Script content: MEDIUM — game logic
- Game session logs: MEDIUM — may contain player interaction data
- Client connection data: MEDIUM — player connection information

## Prohibited Actions

- NEVER use Realtime Servers for compute-intensive game logic (use custom servers)
- NEVER deploy untested script updates to production
- NEVER include secrets in JavaScript scripts
- NEVER confuse Realtime Server scripts with custom game server builds

## Phase 3 — Rollback

1. If script update caused issues: `aws gamelift update-script --script-id <id> --zip-file fileb://previous-version.zip`
2. If fleet was created with wrong script: create new fleet with correct script
3. If client SDK was updated: revert client to compatible version
4. Verify game sessions function correctly after rollback
5. Document script changes and update testing procedures

## Common Issues

- symptoms: "Realtime Server game sessions start but clients cannot connect"
  diagnosis: "Script does not implement onPlayerConnect callback or port configuration is incorrect."
  resolution: "Implement onPlayerConnect callback in script. Verify fleet port settings match client connection configuration."

## Output Format

```yaml
root_cause: "realtime_servers — <specific_cause>"
evidence:
  - type: realtime_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Realtime Server script"
  long_term: "Implement script testing framework and versioned deployments"
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
