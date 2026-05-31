---
title: "B1 — Profiler Agent Setup Issues"
description: "Diagnose profiler agent installation and configuration"
status: active
severity: HIGH
triggers:
  - "profiler not collecting data"
  - "agent not starting"
  - "no profiling data"
owner: devops-agent
objective: "Resolve profiler agent setup issues"
context: "Profiler agent must be integrated into the application runtime (Java/Python)."
---

## Phase 1 — Triage

MUST:
- Verify agent library is included in application
- Check profiling group exists: `aws codeguru-profiler list-profiling-groups`
- Verify IAM role has codeguru-profiler permissions
- Check application logs for agent errors

SHOULD:
- Verify agent configuration (profiling group name, region)
- Check network connectivity to CodeGuru endpoints
- Verify agent version is current

## Phase 2 — Remediate

MUST:
- Add agent library to application dependencies
- Create profiling group if missing
- Fix IAM permissions

SHOULD:
- Configure agent with correct profiling group name
- Test in non-production first

## Common Issues

- symptoms: "Profiling group shows no data"
  diagnosis: "Agent not integrated or not configured correctly."
  resolution: "Add agent to application. Configure profiling group name. Redeploy."

## Output Format

```yaml
root_cause: "profiler_agent — <specific_cause>"
evidence:
  - type: agent_status
    content: "<agent configuration>"
severity: HIGH
mitigation:
  immediate: "Fix agent integration"
  long_term: "Automate agent deployment"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Add agent library to application dependencies | YELLOW |
| Create profiling group if missing | GREEN |
| Fix IAM permissions | YELLOW |
| Configure agent with correct profiling group name | GREEN |
| Test in non-production first | GREEN |

## Escalation Conditions

- Fix requires modifying production application dependencies
- Fix involves deploying agent changes to production runtime

## Data Sensitivity

- **HIGH**: profiling data contains application runtime behavior and performance characteristics
- Agent configuration may expose application architecture and deployment details

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If agent library was added, remove from application dependencies and redeploy
2. If profiling group was created, delete if not needed: `aws codeguru-profiler delete-profiling-group --profiling-group-name <name>`
3. If IAM permissions were modified, revert to previous policy
4. Verify rollback by confirming application runs correctly without profiler agent

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
