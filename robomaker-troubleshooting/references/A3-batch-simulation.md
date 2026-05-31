---
title: "A3 — Batch Simulation Issues"
description: "Diagnose batch simulation job failures and configuration"
status: active
severity: MEDIUM
triggers:
  - "batch simulation failed"
  - "parallel simulation error"
  - "batch job timeout"
owner: devops-agent
objective: "Resolve batch simulation job issues"
context: "Batch simulation runs multiple simulation jobs in parallel with different configurations."
---

## Phase 1 — Triage

MUST:
- List batch jobs: `aws robomaker list-simulation-job-batches`
- Describe batch: `aws robomaker describe-simulation-job-batch --batch <batch-arn>`
- Check individual job statuses within the batch
- Verify concurrent job limits

SHOULD:
- Check for common failure patterns across batch jobs
- Verify batch configuration parameters
- Check S3 output paths for conflicts

MAY:
- Review batch job scheduling and resource allocation
- Check for throttling on batch submission

## Phase 2 — Remediate

MUST:
- Fix common failure causes across batch jobs
- Resubmit failed batch or individual jobs
- Request quota increase if hitting limits

SHOULD:
- Stagger batch submissions to avoid throttling
- Use unique output paths per job

## Common Issues

- symptoms: "Some batch jobs fail with LimitExceededException"
  diagnosis: "Concurrent simulation job limit exceeded."
  resolution: "Reduce batch size or request quota increase."

## Output Format

```yaml
root_cause: "batch_simulation — <specific_cause>"
evidence:
  - type: batch_status
    content: "<batch job status>"
severity: MEDIUM
mitigation:
  immediate: "Fix failures and resubmit"
  long_term: "Right-size batch configurations"
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
