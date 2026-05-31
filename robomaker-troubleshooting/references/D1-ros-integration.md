---
title: "D1 — ROS Integration Issues"
description: "Diagnose ROS version compatibility and integration problems"
status: active
severity: HIGH
triggers:
  - "ROS version mismatch"
  - "ROS bridge error"
  - "catkin build failed"
  - "colcon build failed"
owner: devops-agent
objective: "Resolve ROS integration and compatibility issues"
context: "ROS1 (Melodic) and ROS2 (Foxy) have different APIs, build systems, and message formats."
---

## Phase 1 — Triage

MUST:
- Check robot application ROS version: `aws robomaker describe-robot-application --application <arn>`
- Check simulation application ROS version
- Verify both use the same ROS version
- Check build system (catkin for ROS1, colcon for ROS2)

SHOULD:
- Verify message type compatibility
- Check for deprecated ROS APIs
- Verify launch file syntax matches ROS version

MAY:
- Check ROS bridge configuration if mixing versions
- Verify custom message definitions

## Phase 2 — Remediate

MUST:
- Align ROS versions between robot and simulation apps
- Fix build system configuration
- Update deprecated API calls

SHOULD:
- Migrate to ROS2 for new projects
- Document ROS version requirements

## Common Issues

- symptoms: "ImportError: No module named 'catkin_pkg'"
  diagnosis: "ROS1 build tools not available in ROS2 environment."
  resolution: "Use colcon build system for ROS2. Update package structure."

## Output Format

```yaml
root_cause: "ros_integration — <specific_cause>"
evidence:
  - type: ros_config
    content: "<ROS version and build details>"
severity: HIGH
mitigation:
  immediate: "Fix ROS version alignment"
  long_term: "Standardize on ROS2"
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
