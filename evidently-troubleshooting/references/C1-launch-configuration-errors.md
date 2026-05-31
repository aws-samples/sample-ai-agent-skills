---
title: "C1 — Launch Configuration Errors"
description: "Diagnose launch configuration and creation failures"
status: active
severity: HIGH
triggers: ["launch creation failed", "launch configuration error", "traffic split wrong", "CreateLaunch error"]
owner: devops-agent
objective: "Resolve launch configuration issues including traffic splits and group definitions"
context: "Launches control gradual feature rollouts with traffic percentage splits across variations. They support scheduled steps for automated rollout progression."
---

## Phase 1 — Triage

MUST:
- List launches in the project: `aws evidently list-launches --project <name>`
- Get launch details: `aws evidently get-launch --project <name> --launch <launch-name>`
- Verify the feature referenced by the launch exists and has the expected variations

SHOULD:
- Check launch status (CREATED, RUNNING, COMPLETED, CANCELLED)
- Verify traffic split percentages sum to 100% per group
- Review scheduled steps configuration

MAY:
- Check CloudTrail for CreateLaunch or UpdateLaunch errors
- Verify the feature's evaluation strategy is compatible with the launch

## Phase 2 — Remediate

MUST:
- Fix traffic split percentages to sum to 100% per group
- Ensure the launch references valid feature variations
- Verify the feature exists in the project

SHOULD:
- Configure scheduled steps for automated rollout progression
- Set appropriate group definitions for traffic segmentation

MAY:
- Add metric monitors to the launch for observability (note: launches don't analyze metrics)
- Configure launch description and tags

## Common Issues

- symptoms: "Launch created but traffic not splitting as expected"
  diagnosis: "Launch may not be started, or entity does not match the launch group."
  resolution: "Verify launch status is RUNNING: `aws evidently get-launch --project <name> --launch <launch-name>`. Start if needed: `aws evidently start-launch --project <name> --launch <launch-name>`."

## Safety Ratings

```
safety_ratings:
  - "list-launches: GREEN — read-only launch enumeration"
  - "get-launch: GREEN — read-only launch details"
  - "start-launch: YELLOW — begins traffic splitting for the feature"
  - "stop-launch: RED — stops traffic splitting, reverts to default variation"
```

## Escalation Conditions

- Launch traffic splits not matching configured percentages after sufficient sample size
- Launch start fails with internal errors
- Scheduled steps not executing at configured times
- Launch affecting production traffic in unexpected ways

## Data Sensitivity

- `aws evidently list-launches`: LOW — returns launch metadata
- `aws evidently get-launch`: LOW — returns launch configuration and traffic splits
- Launch traffic data: MEDIUM — reveals feature rollout strategy

## Prohibited Actions

- NEVER stop a production launch without explicit approval from the feature owner
- NEVER modify traffic splits on a running launch without understanding the impact
- NEVER delete a launch that is currently RUNNING
- NEVER assume launches collect metrics — they only control traffic allocation

## Phase 3 — Rollback

1. Stop the launch if it was started in error: `aws evidently stop-launch --project <name> --launch <launch-name> --desired-state CANCELLED`
2. Revert traffic split changes by updating the launch configuration
3. If scheduled steps were modified, restore original schedule
4. Verify feature evaluations return to default variation after launch stop
5. Document the launch configuration issue and resolution

## Output Format

```yaml
root_cause: "launch_configuration — <specific_cause>"
evidence:
  - type: configuration_error
    content: "<launch configuration details>"
severity: HIGH
mitigation:
  immediate: "Fix launch configuration"
  long_term: "Define launches in IaC with validated traffic splits"
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
