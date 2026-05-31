---
title: "C2 — Robot Registration Issues"
description: "Diagnose robot registration and identity problems"
status: active
severity: MEDIUM
triggers:
  - "robot registration failed"
  - "robot not appearing"
  - "robot status error"
owner: devops-agent
objective: "Resolve robot registration issues"
context: "Robots must be registered with RoboMaker and have Greengrass Core for fleet deployment."
---

## Phase 1 — Triage

MUST:
- List robots: `aws robomaker list-robots`
- Describe robot: `aws robomaker describe-robot --robot <robot-arn>`
- Check Greengrass Core status on the robot
- Verify IoT thing and certificate

SHOULD:
- Check robot architecture matches application
- Verify network connectivity from robot to AWS
- Check IAM role attached to Greengrass Core

MAY:
- Verify robot hardware meets requirements
- Check for duplicate registrations

## Phase 2 — Remediate

MUST:
- Install and configure Greengrass Core on robot
- Register robot with correct architecture
- Ensure valid IoT certificates

SHOULD:
- Add robot to appropriate fleet
- Configure robot-specific environment variables

## Common Issues

- symptoms: "Robot shows FAILED status"
  diagnosis: "Greengrass Core not running or certificates invalid."
  resolution: "Restart Greengrass Core. Verify IoT certificates are valid."

## Output Format

```yaml
root_cause: "robot_registration — <specific_cause>"
evidence:
  - type: robot_status
    content: "<robot registration status>"
severity: MEDIUM
mitigation:
  immediate: "Fix registration issue"
  long_term: "Automate robot provisioning"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Simulation configurations may contain proprietary robot designs
- Fleet management data reveals operational robot deployments
- ROS topic data may contain sensor readings and telemetry
- Classification: CONFIDENTIAL - proprietary robotics IP and operational data

## Prohibited Actions

- NEVER terminate running simulation jobs without confirming no dependent processes
- NEVER delete robot applications that are deployed to physical robots
- NEVER modify fleet configurations during active deployments
- NEVER remove Greengrass deployment targets without verifying robot state

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
