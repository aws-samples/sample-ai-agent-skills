---
title: "D2 — Feature Flag Toggle Issues"
description: "Diagnose feature flag toggle and evaluation problems"
status: active
severity: HIGH
triggers:
  - "flag not toggling"
  - "flag stuck"
  - "wrong flag value"
owner: devops-agent
objective: "Resolve feature flag toggle and evaluation issues"
context: "Feature flags are evaluated by the AppConfig agent/extension. Toggle changes require a new deployment."
---

## Phase 1 — Triage

MUST:
- Check current deployed configuration version in the target environment
- Verify the flag enabled state in the deployed configuration
- Check if a deployment is in progress for the flag change
- Verify the AppConfig agent/extension is running on the client

SHOULD:
- Check the AppConfig agent cache TTL configuration
- Verify the application is calling the correct AppConfig endpoint

MAY:
- Enable debug logging on the AppConfig agent

## Phase 2 — Remediate

MUST:
- Deploy the updated flag configuration if not yet deployed
- Verify the deployment completed successfully
- Wait for agent cache TTL to expire for clients to pick up changes

SHOULD:
- Restart the AppConfig agent if cache appears stale
- Verify the application is parsing the flag response correctly

MAY:
- Reduce cache TTL for faster flag propagation (trade-off: more API calls)

safety_ratings:
  - "Checking flag state: GREEN"
  - "Deploying flag change: YELLOW"
  - "Restarting AppConfig agent: YELLOW"

escalation_conditions:
  - "Production feature flag stuck in wrong state after deployment"
  - "Flag toggle causing application errors"

data_sensitivity:
  - command: "get-latest-configuration"
    sensitivity: MEDIUM
    contains: "Current flag values and targeting rules"

prohibited_actions:
  - "NEVER force-kill the AppConfig agent to refresh cache in production"

## Phase 3 — Rollback

- Deploy the previous configuration version to revert the flag toggle
- Verify clients receive the reverted flag value after cache refresh

## Common Issues

- symptoms: "Flag toggled but application still sees old value"
  diagnosis: "AppConfig agent cache hasn't expired yet"
  resolution: "Wait for cache TTL or restart the agent to force refresh"

- symptoms: "Flag toggle deployed but some instances see different values"
  diagnosis: "Deployment still in progress (partial rollout per strategy)"
  resolution: "Wait for deployment to complete across all targets"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: flag_state
    content: "<current vs expected flag value>"
severity: HIGH
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
