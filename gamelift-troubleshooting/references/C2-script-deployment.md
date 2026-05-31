---
title: "C2 — Script Deployment"
description: "Diagnose GameLift Realtime Servers script deployment issues"
status: active
severity: MEDIUM
triggers: ["script upload failed", "Realtime script error", "script deployment", "script not working"]
owner: devops-agent
objective: "Resolve Realtime Servers script upload and deployment issues"
context: "Scripts are JavaScript files for Realtime Servers. They are different from builds (compiled executables). Scripts can be updated on existing fleets unlike builds."
---

## Phase 1 — Triage

MUST:
- Check script status: `aws gamelift describe-script --script-id <script-id>`
- List scripts: `aws gamelift list-scripts`
- Verify script file is valid JavaScript

SHOULD:
- Check if script implements required callback functions (onStartGameSession, onProcessStarted)
- Verify S3 permissions if using S3-based script storage

MAY:
- Review fleet events for script-related errors: `aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>`
- Test script logic locally before deployment

## Phase 2 — Remediate

MUST:
- Upload script: `aws gamelift create-script --name <name> --script-version <version> --zip-file fileb://script.zip`
- Or from S3: `aws gamelift create-script --name <name> --storage-location Bucket=<bucket>,Key=<key>,RoleArn=<role-arn>`
- Ensure script implements required Realtime Server callbacks

SHOULD:
- Update existing script: `aws gamelift update-script --script-id <script-id> --zip-file fileb://updated-script.zip`
- Include all required JavaScript modules in the zip file

MAY:
- Use script versioning for rollback capability
- Implement logging in scripts for debugging

## Safety Ratings

```
safety_ratings:
  - "describe-script / list-scripts: GREEN — read-only inspection"
  - "create-script: YELLOW — uploads new script"
  - "update-script: YELLOW — updates script on existing fleets (rolling update)"
  - "delete-script: RED — removes script, affects fleets using it"
```

## Escalation Conditions

- Script update causes fleet instability
- JavaScript runtime errors in Realtime Server callbacks
- Script size exceeds limits
- S3 access issues for script storage

## Data Sensitivity

- `aws gamelift describe-script`: LOW — script metadata
- Script content: MEDIUM — contains game logic
- S3 script storage: MEDIUM — contains game server logic
- `aws cloudtrail lookup-events`: MEDIUM — deployment activity

## Prohibited Actions

- NEVER delete scripts referenced by active fleets
- NEVER deploy untested scripts to production fleets
- NEVER include credentials or secrets in script files
- NEVER use scripts for custom game servers (use builds instead)

## Phase 3 — Rollback

1. If script update caused issues: `aws gamelift update-script --script-id <id> --zip-file fileb://previous-version.zip`
2. If script was deleted: recreate from source
3. If fleet was created with wrong script: update script or create new fleet
4. Verify game sessions are functioning after rollback
5. Document script changes and update deployment procedures

## Common Issues

- symptoms: "Realtime Server fleet fails to activate after script deployment"
  diagnosis: "Script does not implement required callback functions or has JavaScript syntax errors."
  resolution: "Verify script implements onStartGameSession and onProcessStarted callbacks. Check for JavaScript syntax errors. Test locally first."

## Output Format

```yaml
root_cause: "script_deployment — <specific_cause>"
evidence:
  - type: script_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix script deployment issue"
  long_term: "Implement script testing and versioned deployment pipeline"
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
