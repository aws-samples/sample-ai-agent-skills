---
title: "C2 — Launch Scheduling Issues"
description: "Diagnose launch scheduling and step execution problems"
status: active
severity: MEDIUM
triggers: ["scheduled step not executing", "launch schedule wrong", "step timing issue", "rollout not progressing"]
owner: devops-agent
objective: "Resolve launch scheduling issues including step execution and timing"
context: "Launches support scheduled steps that automatically change traffic allocation at specified times. Steps execute in order and cannot be skipped."
---

## Phase 1 — Triage

MUST:
- Get launch details with schedule: `aws evidently get-launch --project <name> --launch <launch-name>`
- Check launch status — must be RUNNING for steps to execute
- Review the scheduled steps configuration and their timestamps

SHOULD:
- Verify step timestamps are in the future (for pending steps) or past (for executed steps)
- Check if the launch was started after the first step's scheduled time
- Verify timezone handling — timestamps should be in UTC

MAY:
- Review CloudTrail for launch-related events
- Check if any manual traffic split updates overrode scheduled steps

## Phase 2 — Remediate

MUST:
- Ensure the launch is in RUNNING status for scheduled steps to execute
- Verify step timestamps are correctly set in UTC
- Confirm steps are ordered chronologically

SHOULD:
- If steps were missed, manually update traffic splits to the intended state
- Reschedule future steps if timing needs adjustment

MAY:
- Create a new launch with corrected schedule if the current one cannot be fixed
- Set up monitoring for step execution events

## Common Issues

- symptoms: "Scheduled step did not execute at the configured time"
  diagnosis: "Launch may not be in RUNNING status, or the step time has not yet arrived (timezone confusion)."
  resolution: "Verify launch status is RUNNING and step timestamps are in UTC. If the launch was started after a step's time, that step is skipped."

## Safety Ratings

```
safety_ratings:
  - "get-launch: GREEN — read-only launch details"
  - "update-launch: YELLOW — modifies launch configuration and schedule"
  - "start-launch: YELLOW — begins traffic splitting"
  - "stop-launch: RED — stops traffic splitting"
```

## Escalation Conditions

- Scheduled steps consistently not executing despite correct configuration
- Launch schedule changes needed during an active production rollout
- Step execution causes unexpected traffic distribution
- Multiple launches with conflicting schedules on the same feature

## Data Sensitivity

- `aws evidently get-launch`: LOW — returns launch configuration and schedule
- Launch schedule: LOW — operational configuration data

## Prohibited Actions

- NEVER modify scheduled steps on a running production launch without stakeholder approval
- NEVER skip steps by manually setting traffic splits without documenting the deviation
- NEVER delete a running launch to fix scheduling issues
- NEVER assume step execution times are in local timezone — they are UTC

## Phase 3 — Rollback

1. If schedule was modified incorrectly, update with correct timestamps: `aws evidently update-launch --project <name> --launch <launch-name> --scheduled-splits-config <config>`
2. If traffic splits were manually overridden, restore to the intended step configuration
3. If the launch needs to restart, stop and create a new launch with correct schedule
4. Document the scheduling issue and any manual interventions
5. Verify traffic distribution matches the intended rollout plan

## Output Format

```yaml
root_cause: "launch_scheduling — <specific_cause>"
evidence:
  - type: scheduling_issue
    content: "<schedule configuration and step execution details>"
severity: MEDIUM
mitigation:
  immediate: "Fix launch schedule or manually adjust traffic splits"
  long_term: "Validate launch schedules in staging before production deployment"
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
