---
title: "B1 — Robot Application Build Errors"
description: "Diagnose robot application compilation and build failures"
status: active
severity: HIGH
triggers:
  - "build failed"
  - "compilation error"
  - "dependency not found"
owner: devops-agent
objective: "Resolve robot application build failures"
context: "Robot applications must be valid ROS packages compiled for the target architecture."
---

## Phase 1 — Triage

MUST:
- Check application details: `aws robomaker describe-robot-application --application <app-arn>`
- Verify ROS version matches (Melodic/Foxy)
- Check target architecture (X86_64/ARM64)
- Review build logs for compilation errors

SHOULD:
- Verify package.xml dependencies are available
- Check CMakeLists.txt or setup.py for errors
- Verify source bundle in S3 is complete

MAY:
- Build locally to reproduce errors
- Check for missing system dependencies

## Phase 2 — Remediate

MUST:
- Fix compilation errors in source code
- Add missing dependencies to package manifest
- Rebuild and upload corrected bundle

SHOULD:
- Test builds locally before uploading
- Pin dependency versions for reproducibility

## Common Issues

- symptoms: "Could not find package <name>"
  diagnosis: "ROS dependency not declared in package.xml."
  resolution: "Add missing dependency to package.xml and rebuild."

## Output Format

```yaml
root_cause: "robot_app_build — <specific_cause>"
evidence:
  - type: build_logs
    content: "<build error details>"
severity: HIGH
mitigation:
  immediate: "Fix build errors and rebuild"
  long_term: "Add CI/CD for application builds"
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
