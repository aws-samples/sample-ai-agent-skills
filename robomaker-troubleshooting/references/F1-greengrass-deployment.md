---
title: "F1 — Greengrass Deployment Issues"
description: "Diagnose fleet deployment via Greengrass"
status: active
severity: HIGH
triggers:
  - "deployment failed"
  - "Greengrass error"
  - "robot not updating"
owner: devops-agent
objective: "Resolve Greengrass-based fleet deployment issues"
context: "Fleet deployment uses IoT Greengrass to push robot applications to physical robots."
---

## Phase 1 — Triage

MUST:
- Check deployment status: `aws robomaker describe-deployment-job --job <job-arn>`
- Verify Greengrass Core is running on target robots
- Check IoT thing connectivity
- Verify IAM role for Greengrass deployment

SHOULD:
- Check Greengrass logs on the robot
- Verify network connectivity from robot to AWS IoT
- Check certificate validity

MAY:
- Verify robot has sufficient disk space
- Check for conflicting Greengrass deployments

## Phase 2 — Remediate

MUST:
- Restart Greengrass Core if not running
- Fix certificate issues
- Ensure network connectivity

SHOULD:
- Update Greengrass Core to latest version
- Configure deployment rollback

## Common Issues

- symptoms: "Deployment stuck in InProgress"
  diagnosis: "Robot cannot download application bundle."
  resolution: "Check robot network connectivity. Verify S3 access from Greengrass role."

## Output Format

```yaml
root_cause: "greengrass_deployment — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<deployment job status>"
severity: HIGH
mitigation:
  immediate: "Fix deployment blockers"
  long_term: "Automate deployment validation"
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
