---
title: "A1 — Simulation Job Failures"
description: "Diagnose simulation job creation and execution failures"
status: active
severity: HIGH
triggers:
  - "simulation job failed"
  - "SimulationJobFailed"
  - "job terminated unexpectedly"
owner: devops-agent
objective: "Resolve simulation job failures in RoboMaker"
context: "Simulation jobs can fail due to application errors, resource limits, VPC issues, or S3 permission problems."
---

## Phase 1 — Triage

MUST:
- Describe job: `aws robomaker describe-simulation-job --job <job-arn>`
- Check failureCode and failureReason fields
- Review CloudWatch Logs: `/aws/robomaker/SimulationJobs/<job-id>`
- Verify IAM role permissions

SHOULD:
- Check S3 bucket access for application bundles
- Verify VPC configuration if using custom VPC
- Check concurrent job limits: `aws service-quotas get-service-quota --service-code robomaker --quota-code L-<id>`

MAY:
- Check for application-level errors in logs
- Verify Gazebo world file syntax

## Phase 2 — Remediate

MUST:
- Fix identified failure cause (permissions, S3, VPC)
- Resubmit simulation job

SHOULD:
- Add error handling to robot application
- Configure appropriate job timeout

## Common Issues

- symptoms: "failureCode: ResourceNotFound"
  diagnosis: "S3 application bundle not found."
  resolution: "Verify S3 path and bucket permissions."

- symptoms: "failureCode: SimulationApplicationCrash"
  diagnosis: "Robot or simulation application crashed during execution."
  resolution: "Check CloudWatch Logs for crash details. Fix application code."

## Output Format

```yaml
root_cause: "simulation_job — <specific_cause>"
evidence:
  - type: job_status
    content: "<failure code and reason>"
severity: HIGH
mitigation:
  immediate: "Fix failure cause and resubmit"
  long_term: "Add monitoring and error handling"
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
