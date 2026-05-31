---
title: "D1 — Recovery Launch Failures"
description: "Diagnose recovery instance launch failures"
status: active
severity: CRITICAL
triggers:
  - "recovery launch failed"
  - "start recovery error"
  - "cannot launch instance"
owner: devops-agent
objective: "Resolve recovery launch failures"
context: "Recovery launches create EC2 instances from replicated data. Failures can occur due to launch config errors, EC2 capacity, IAM permissions, or snapshot issues."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws drs describe-jobs --filters 'fromDate=2024-01-01T00:00:00Z' --query 'items[?type==LAUNCH].{Id:jobID,Status:status,Created:creationDateTime}'`
- Get job log details: `aws drs describe-job-log-items --job-id <job-id>`
- Verify source server state is READY_FOR_TEST or READY_FOR_CUTOVER
- Check launch configuration: `aws drs get-launch-configuration --source-server-id <id>`

SHOULD:
- Verify EC2 instance limits in target account
- Check IAM role permissions for DRS service role
- Verify target subnet and security group validity

MAY:
- Check for EC2 capacity issues in the target AZ
- Review CloudTrail for RunInstances failures

## Phase 2 — Remediate

MUST:
- Fix launch configuration issues identified in job logs
- Ensure DRS service role has required EC2 permissions
- Verify source server lifecycle state allows launch

SHOULD:
- Retry launch after fixing configuration
- Test with drill launch first

MAY:
- Try alternative AZ if capacity is the issue

## Common Issues

- symptoms: "Job fails with LAUNCH_FAILED status"
  diagnosis: "Check job log items for specific EC2 launch error."
  resolution: "Fix the specific error (IAM, capacity, config) and retry."

## Output Format

```yaml
root_cause: "recovery_launch — <specific_cause>"
evidence:
  - type: job_logs
    content: "<job log error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix launch blocker and retry"
  long_term: "Regular drill testing to catch launch issues proactively"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Recovery launches create instances from replicated production data |
| Blast Radius | CRITICAL — Launch failures during actual disaster mean workloads cannot be recovered |
| Rollback Complexity | MEDIUM — Fix configuration and retry; but time is critical during actual DR |
| Compliance Scope | CRITICAL — Recovery capability is the core DR compliance requirement |

## Escalation Conditions

- Escalate IMMEDIATELY to DR coordinator if recovery launch fails during actual disaster
- Escalate to operations if launch configuration issues are discovered during drills
- Escalate to AWS Support if launches fail with internal errors

## Data Sensitivity

- **Recovery instances** contain production data — apply same security controls
- **Launch configuration** reveals DR architecture
- **Job logs** may contain infrastructure details

## PROHIBITED Actions

- Do NOT attempt recovery launches without verified launch configurations
- Do NOT skip drill testing — it catches launch issues before actual disasters
- Do NOT launch recovery instances in unsecured subnets or security groups

## Phase 3 — Rollback

- Fix the launch configuration issue identified in job logs
- Retry the recovery launch: `aws drs start-recovery --source-servers sourceServerID=<id>`
- If recovery instance was launched incorrectly, terminate and relaunch
- Document the failure for DR readiness improvement

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
