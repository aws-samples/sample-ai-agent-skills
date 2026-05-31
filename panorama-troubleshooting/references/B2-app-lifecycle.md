---
title: "B2 — Application Lifecycle Issues"
description: "Diagnose running application crashes, restarts, and removal"
status: active
severity: MEDIUM
triggers:
  - "application crashed"
  - "app keeps restarting"
  - "cannot remove application"
owner: devops-agent
objective: "Resolve application runtime and lifecycle issues"
context: "Running applications can crash due to code errors, resource exhaustion, or camera disconnections."
---

## Phase 1 — Triage

MUST:
- Check app health: `aws panorama describe-application-instance --application-instance-id <id>`
- Review application logs in CloudWatch: `/aws/panorama/devices/<device-id>`
- Check device resource utilization via CloudWatch metrics
- Verify camera streams are still connected

SHOULD:
- Check for OOM (out of memory) errors in logs
- Verify application code handles exceptions properly
- Check if multiple apps are competing for GPU resources

MAY:
- Review application node graph for configuration issues
- Check for firmware compatibility issues

## Phase 2 — Remediate

MUST:
- Fix application code errors identified in logs
- Restart application if in error state
- Remove and redeploy if application is unrecoverable

SHOULD:
- Reduce concurrent applications if resource-constrained
- Add error handling to application code

## Common Issues

- symptoms: "Application status REMOVAL_FAILED"
  diagnosis: "Application resources not properly cleaned up."
  resolution: "Force remove: `aws panorama remove-application-instance --application-instance-id <id>`"

## Output Format

```yaml
root_cause: "app_lifecycle — <specific_cause>"
evidence:
  - type: app_status
    content: "<application runtime status>"
severity: MEDIUM
mitigation:
  immediate: "Restart or redeploy application"
  long_term: "Add monitoring and error handling"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — running applications process video streams
- Service Disruption Risk: MEDIUM — lifecycle issues affect application availability
- Compliance Impact: MEDIUM — application uptime affects monitoring coverage

## Escalation Conditions
- Application crashes repeatedly without recovery
- Application memory or CPU usage grows unbounded
- Application stops processing video streams
- Application lifecycle events not being reported
- Multiple applications on same device interfere with each other

## Data Sensitivity
- Application runtime logs — INTERNAL (may contain CONFIDENTIAL data)
- Video processing metrics — INTERNAL
- Application health data — INTERNAL
- Resource utilization metrics — INTERNAL
- Never expose video stream data in application logs

## Prohibited Actions
- Do NOT force-stop applications without understanding impact on monitoring
- Do NOT ignore application health degradation
- Do NOT run applications beyond device resource capacity
- Do NOT bypass application lifecycle management procedures
- Do NOT disable application logging on production devices
- Do NOT modify running application configurations without testing

## Phase 3 — Rollback
- Restart failed applications
- Roll back to previous application version if current version is unstable
- Restore application configurations to previous state
- Verify application health after restart
- Re-enable monitoring if disabled during troubleshooting
- Document application lifecycle issues and resolutions

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
