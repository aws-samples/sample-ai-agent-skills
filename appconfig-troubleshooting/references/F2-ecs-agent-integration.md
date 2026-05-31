---
title: "F2 — ECS Agent Integration"
description: "Diagnose AppConfig agent issues in ECS tasks"
status: active
severity: HIGH
triggers:
  - "ecs appconfig agent"
  - "configuration not available in ecs"
  - "appconfig sidecar"
owner: devops-agent
objective: "Resolve AppConfig agent integration with ECS"
context: "The AppConfig agent runs as a sidecar container in ECS tasks, providing configuration via a local HTTP endpoint."
---

## Phase 1 — Triage

MUST:
- Check ECS task definition for the AppConfig agent sidecar container
- Verify the agent container is running: `aws ecs describe-tasks --cluster <cluster> --tasks <task-id>`
- Check agent container logs for errors
- Verify IAM task role has appconfig:GetLatestConfiguration and appconfig:StartConfigurationSession

SHOULD:
- Verify the agent container port mapping (default 2772)
- Check container dependency ordering (agent should start before app)

MAY:
- Check agent environment variables for configuration

## Phase 2 — Remediate

MUST:
- Add the AppConfig agent sidecar if missing from task definition
- Fix IAM task role permissions
- Ensure container dependency ordering is correct

SHOULD:
- Set the agent as an essential container with health check
- Configure appropriate poll interval

MAY:
- Use ECS service connect for agent communication

safety_ratings:
  - "Checking task definition: GREEN"
  - "Updating task definition: YELLOW"
  - "Modifying IAM task role: YELLOW"

escalation_conditions:
  - "Production ECS service cannot retrieve configuration"
  - "Agent sidecar causing task failures"

data_sensitivity:
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Task configuration, container status, IAM role"

prohibited_actions:
  - "NEVER grant appconfig:* to ECS task roles"

## Phase 3 — Rollback

- If task definition was updated, revert to previous revision
- If IAM role was modified, restore previous policy

## Common Issues

- symptoms: "Application container starts before AppConfig agent is ready"
  diagnosis: "Missing container dependency in task definition"
  resolution: "Add dependsOn with condition HEALTHY for the AppConfig agent container"

- symptoms: "Agent container keeps restarting"
  diagnosis: "IAM permissions missing or configuration path incorrect"
  resolution: "Fix IAM task role and verify agent environment variables"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: task_config
    content: "<ECS task details>"
severity: HIGH
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
