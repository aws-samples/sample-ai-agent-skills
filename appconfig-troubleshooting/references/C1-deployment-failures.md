---
title: "C1 — Deployment Failures"
description: "Diagnose and resolve AppConfig deployment failures"
status: active
severity: CRITICAL
triggers:
  - "deployment failed"
  - "deployment rolled back"
  - "ROLLED_BACK state"
owner: devops-agent
objective: "Identify why a deployment failed and successfully deploy configuration"
context: "Deployments roll out configuration changes according to a deployment strategy. Failures can occur from validation, monitors, or internal errors."
---

## Phase 1 — Triage

MUST:
- Get deployment details: `aws appconfig get-deployment --application-id <app-id> --environment-id <env-id> --deployment-number <num>`
- Check deployment state (BAKING, DEPLOYING, COMPLETE, ROLLED_BACK)
- Check if rollback was triggered by a CloudWatch alarm monitor
- Review the deployment event log for error details

SHOULD:
- Check the deployment strategy used: `aws appconfig get-deployment-strategy --deployment-strategy-id <id>`
- Verify the configuration version being deployed is valid
- Check CloudWatch alarms referenced as environment monitors

MAY:
- Review CloudTrail for StartDeployment and StopDeployment events
- Check if concurrent deployments were attempted

## Phase 2 — Remediate

MUST:
- If rolled back by monitor: investigate the CloudWatch alarm that triggered rollback
- If validation failed: fix configuration content (see B2 runbook)
- If internal error: retry the deployment

SHOULD:
- Use a slower deployment strategy for risky changes (Linear 10% over 10 minutes)
- Ensure monitors are correctly calibrated (not too sensitive)

MAY:
- Create a custom deployment strategy for the specific use case

safety_ratings:
  - "Checking deployment status: GREEN"
  - "Retrying deployment: YELLOW"
  - "Changing deployment strategy: YELLOW"

escalation_conditions:
  - "Production deployment rolled back and root cause unclear"
  - "Multiple consecutive deployment failures"
  - "Deployment stuck in DEPLOYING state"

data_sensitivity:
  - command: "get-deployment"
    sensitivity: MEDIUM
    contains: "Configuration version, deployment state, strategy details"

prohibited_actions:
  - "NEVER force a deployment by removing monitors from a production environment"
  - "NEVER use AllAtOnce strategy for production deployments without approval"

## Phase 3 — Rollback

- AppConfig handles rollback automatically when monitors trigger
- For manual rollback: deploy the previous known-good configuration version
- Document the failed configuration version for investigation

## Common Issues

- symptoms: "Deployment immediately rolls back"
  diagnosis: "CloudWatch alarm monitor was already in ALARM state before deployment"
  resolution: "Resolve the alarm condition first, then retry deployment"

- symptoms: "Deployment stuck in DEPLOYING state"
  diagnosis: "Deployment strategy has a long bake time or growth factor"
  resolution: "Wait for the strategy to complete, or stop the deployment if needed"

- symptoms: "Deployment completes but clients don't see new config"
  diagnosis: "AppConfig agent cache TTL hasn't expired on clients"
  resolution: "Wait for cache TTL to expire, or restart the AppConfig agent/extension"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: deployment_status
    content: "<deployment details>"
severity: CRITICAL
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
