---
title: "F1 — Drill Launch Failures"
description: "Diagnose DR drill test launch failures"
status: active
severity: HIGH
triggers:
  - "drill failed"
  - "test recovery failed"
  - "drill launch error"
owner: devops-agent
objective: "Resolve drill test launch failures"
context: "Drill tests validate DR readiness by launching recovery instances without affecting replication. Drill failures indicate issues that would also affect actual recovery."
---

## Phase 1 — Triage

MUST:
- Check drill job status: `aws drs describe-jobs --filters 'fromDate=2024-01-01T00:00:00Z' --query 'items[?type==LAUNCH && participatingServers[0].launchStatus==FAILED]'`
- Get drill job logs: `aws drs describe-job-log-items --job-id <job-id>`
- Verify source server state is READY_FOR_TEST

SHOULD:
- Check launch configuration for the source server
- Verify EC2 capacity in target AZ
- Check IAM permissions

MAY:
- Review previous successful drill jobs for comparison
- Check EC2 service quotas

## Phase 2 — Remediate

MUST:
- Fix issues identified in drill job logs
- Ensure source server is in READY_FOR_TEST state
- Verify launch configuration is valid

SHOULD:
- Run drills regularly to catch issues early
- Document drill results and remediation steps

MAY:
- Automate drill scheduling with EventBridge

## Common Issues

- symptoms: "Drill fails but source server shows READY_FOR_TEST"
  diagnosis: "Launch configuration issue — invalid subnet, SG, or instance type."
  resolution: "Review and fix launch configuration. Check job logs for specific error."

## Output Format

```yaml
root_cause: "drill_launch — <specific_cause>"
evidence:
  - type: drill_job_logs
    content: "<drill job error details>"
severity: HIGH
mitigation:
  immediate: "Fix launch configuration and retry drill"
  long_term: "Schedule regular drills and automate validation"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Drill instances contain replicated production data |
| Blast Radius | LOW — Drill failures don't affect production or replication |
| Rollback Complexity | LOW — Fix configuration and retry drill |
| Compliance Scope | HIGH — Regular successful drills are required for DR compliance |

## Escalation Conditions

- Escalate to DR coordinator if drill failures indicate DR readiness issues
- Escalate to operations if launch configuration problems are discovered

## Data Sensitivity

- **Drill instances** contain production data — apply same security controls
- **Drill results** document DR readiness — compliance-relevant

## PROHIBITED Actions

- Do NOT skip drill testing — it's the only way to validate DR readiness
- Do NOT leave drill instances running after testing (they incur costs)
- Do NOT ignore drill failures — they predict actual recovery failures

## Phase 3 — Rollback

- Fix the launch configuration issue identified in drill job logs
- Retry the drill after fixing the configuration
- Document drill results for DR compliance records

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
