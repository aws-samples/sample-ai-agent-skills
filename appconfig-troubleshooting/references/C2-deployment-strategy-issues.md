---
title: "C2 — Deployment Strategy Issues"
description: "Diagnose deployment strategy configuration and behavior problems"
status: active
severity: HIGH
triggers:
  - "deployment too slow"
  - "deployment too fast"
  - "strategy not found"
owner: devops-agent
objective: "Configure appropriate deployment strategies"
context: "Deployment strategies control rollout speed via growth type (Linear/Exponential), growth factor, deployment duration, and final bake time."
---

## Phase 1 — Triage

MUST:
- List strategies: `aws appconfig list-deployment-strategies`
- Get strategy details: `aws appconfig get-deployment-strategy --deployment-strategy-id <id>`
- Check growth type (LINEAR or EXPONENTIAL), growth factor, and deployment duration
- Verify the final bake time setting

SHOULD:
- Compare the strategy settings with the deployment behavior observed
- Check if predefined strategies (AllAtOnce, Linear50PercentEvery30Seconds, etc.) meet needs

MAY:
- Review deployment history to see which strategies were used

## Phase 2 — Remediate

MUST:
- Create a new strategy if current one doesn't meet requirements (strategies are immutable)
- Use appropriate growth type: Linear for predictable rollout, Exponential for faster completion

SHOULD:
- Set final bake time > 0 for production deployments to allow monitoring
- Use Linear with small growth factor for high-risk changes

MAY:
- Create environment-specific strategies (fast for dev, slow for prod)

safety_ratings:
  - "Listing strategies: GREEN"
  - "Creating new strategy: GREEN"
  - "Deleting unused strategy: YELLOW"

escalation_conditions:
  - "No suitable strategy exists for a critical production deployment"

data_sensitivity:
  - command: "list-deployment-strategies"
    sensitivity: LOW
    contains: "Strategy names, growth settings"

prohibited_actions:
  - "NEVER use AllAtOnce (0 minute) strategy for production without explicit approval"
  - "NEVER delete a strategy that is referenced by active deployments"

## Phase 3 — Rollback

- Strategies are immutable — no rollback needed for strategy creation
- If wrong strategy was used for a deployment, stop the deployment and redeploy with correct strategy

## Common Issues

- symptoms: "Deployment takes too long"
  diagnosis: "Strategy has a long deployment duration or small growth factor"
  resolution: "Create a new strategy with larger growth factor or shorter duration"

- symptoms: "Deployment completes but issues found after"
  diagnosis: "Final bake time was 0 — no monitoring window after full rollout"
  resolution: "Create a strategy with final bake time of at least 5-10 minutes"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: strategy_config
    content: "<strategy details>"
severity: HIGH
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
