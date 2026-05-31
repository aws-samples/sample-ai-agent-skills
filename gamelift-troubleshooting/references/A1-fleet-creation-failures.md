---
title: "A1 — Fleet Creation Failures"
description: "Diagnose GameLift fleet creation failures"
status: active
severity: HIGH
triggers: ["fleet creation failed", "fleet stuck in ACTIVATING", "fleet ERROR state", "cannot create fleet"]
owner: devops-agent
objective: "Resolve fleet creation and activation failures"
context: "Fleets require a valid build/script, runtime configuration, and IAM role. Fleet transitions: NEW → DOWNLOADING → VALIDATING → BUILDING → ACTIVATING → ACTIVE. Failure at any stage puts fleet in ERROR."
---

## Phase 1 — Triage

MUST:
- Check fleet status and events: `aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>`
- Verify build status: `aws gamelift describe-build --build-id <build-id>`
- Check fleet attributes: `aws gamelift describe-fleet-attributes --fleet-ids <fleet-id>`

SHOULD:
- Verify runtime configuration: check launch path, parameters, and concurrent executions
- Check IAM role permissions: `aws iam get-role --role-name <gamelift-role>`
- Verify instance type is available in the region

MAY:
- Check service quotas for fleet limits: `aws service-quotas get-service-quota --service-code gamelift --quota-code L-4E825B5E`
- Review CloudTrail for CreateFleet errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateFleet --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix build issues: ensure build is in READY status before creating fleet
- Correct runtime configuration: verify launch path matches the executable in the build
- Ensure server process calls GameLift Server SDK ProcessReady() on startup

SHOULD:
- Test build locally before uploading to GameLift
- Use fleet events to identify the exact failure stage
- Verify security group and port configuration match server requirements

MAY:
- Enable remote access for debugging: include EC2 key pair in fleet creation
- Use Anywhere fleets for development and testing

## Safety Ratings

```
safety_ratings:
  - "describe-fleet-events / describe-build: GREEN — read-only inspection"
  - "create-fleet: YELLOW — creates EC2 instances, incurs costs"
  - "delete-fleet: RED — terminates all instances and game sessions"
  - "update-fleet-attributes: YELLOW — modifies fleet configuration"
```

## Escalation Conditions

- Fleet stuck in ACTIVATING for more than 30 minutes
- Build validation fails with no clear error message
- Instance type not available in the target region
- Service quota for fleets reached

## Data Sensitivity

- `aws gamelift describe-fleet-events`: LOW — fleet lifecycle events
- `aws gamelift describe-build`: LOW — build metadata
- `aws iam get-role`: MEDIUM — role configuration
- Fleet creation parameters: MEDIUM — may contain server configuration details

## Prohibited Actions

- NEVER create fleets with untested builds in production
- NEVER use overly large instance types for testing (cost implications)
- NEVER delete fleets with active game sessions without player notification
- NEVER skip Server SDK integration in custom game servers

## Phase 3 — Rollback

1. If fleet was created with wrong build: delete fleet and create with correct build
2. If runtime configuration was wrong: `aws gamelift update-runtime-configuration --fleet-id <id> --runtime-configuration <config>` (only works on ACTIVE fleets)
3. If fleet is stuck in ERROR: delete and recreate after fixing the root cause
4. If wrong instance type was used: delete fleet and create with correct type
5. Document the failure and update build/deployment procedures

## Common Issues

- symptoms: "Fleet stuck in ACTIVATING state and eventually moves to ERROR"
  diagnosis: "Server process is not calling GameLift Server SDK ProcessReady() within the timeout period."
  resolution: "Verify server binary integrates GameLift Server SDK. Ensure ProcessReady() is called on startup. Check server logs for crashes."

## Output Format

```yaml
root_cause: "fleet_creation — <specific_cause>"
evidence:
  - type: fleet_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix fleet creation blocker"
  long_term: "Implement build validation pipeline before fleet creation"
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
