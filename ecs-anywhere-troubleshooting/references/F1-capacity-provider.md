---
title: "F1 — Capacity Provider"
description: "Diagnose capacity provider configuration issues for ECS Anywhere"
status: active
severity: MEDIUM
triggers: ["capacity provider error", "no capacity", "capacity strategy failed", "provider not found"]
owner: devops-agent
objective: "Resolve capacity provider configuration issues for ECS Anywhere"
context: "ECS Anywhere uses a default capacity provider for external instances. Custom capacity providers can be configured but auto-scaling of external instances is not supported."
---

## Phase 1 — Triage

MUST:
- Check cluster capacity providers: `aws ecs describe-clusters --clusters <cluster-name> --include ATTACHMENTS --query 'clusters[*].{Providers:capacityProviders,DefaultStrategy:defaultCapacityProviderStrategy}'`
- Check capacity provider details: `aws ecs describe-capacity-providers --capacity-providers <provider-name>`
- Verify external instances are registered and active: `aws ecs list-container-instances --cluster <cluster-name> --status ACTIVE`

SHOULD:
- Check if the service specifies a capacity provider strategy
- Verify the capacity provider is associated with the cluster
- Check available resources across registered instances

MAY:
- Review service events for capacity-related errors
- Check if multiple capacity providers are configured and their weights

## Phase 2 — Remediate

MUST:
- Associate the capacity provider with the cluster if missing
- Ensure the service or task uses the correct capacity provider strategy
- Register additional external instances if capacity is insufficient

SHOULD:
- Configure default capacity provider strategy for the cluster
- Set appropriate weights if using multiple capacity providers
- Monitor capacity utilization across external instances

MAY:
- Implement alerting for low capacity situations
- Create automation for instance registration when capacity is low
- Use capacity provider strategies to balance across instance groups

## Common Issues

- symptoms: "Service cannot place tasks — 'no capacity provider found'"
  diagnosis: "Capacity provider not associated with the cluster, or service uses wrong provider strategy."
  resolution: "Associate the capacity provider with the cluster. Update service to use the correct capacity provider strategy."

- symptoms: "Tasks placed unevenly across external instances"
  diagnosis: "Capacity provider strategy weights are not balanced, or placement constraints are too restrictive."
  resolution: "Adjust capacity provider strategy weights. Review placement constraints."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-clusters: GREEN — read-only"
  - "ecs describe-capacity-providers: GREEN — read-only"
  - "ecs list-container-instances: GREEN — read-only"
  - "ecs put-cluster-capacity-providers: YELLOW — modifies cluster capacity configuration"
  - "ecs update-service: YELLOW — changes service capacity strategy"
```

## Escalation Conditions

- No external instances available and new instances cannot be registered quickly
- Capacity provider configuration changes affect other services on the cluster
- Auto-scaling is required but not supported for external instances
- Capacity planning requires infrastructure procurement

## Data Sensitivity

- `aws ecs describe-clusters`: LOW — cluster configuration
- `aws ecs describe-capacity-providers`: LOW — provider configuration
- `aws ecs list-container-instances`: LOW — instance ARNs
- Capacity utilization data: LOW — resource usage metrics

## Prohibited Actions

- NEVER remove capacity providers from a cluster while services depend on them
- NEVER assume ECS will auto-scale external instances — it cannot
- NEVER set capacity provider weights to 0 for active providers
- NEVER modify capacity provider strategy during active deployments

## Phase 3 — Rollback

1. If capacity provider association was changed, restore previous configuration
2. If service capacity strategy was modified, revert: `aws ecs update-service --cluster <cluster-name> --service <service-name> --capacity-provider-strategy <previous-strategy>`
3. If default cluster strategy was changed, restore it
4. Verify task placement works correctly after rollback
5. Document the capacity provider architecture

## Output Format

```yaml
root_cause: "capacity_provider — <specific_cause>"
evidence:
  - type: capacity_error
    content: "<capacity provider and resource details>"
severity: MEDIUM
mitigation:
  immediate: "Fix capacity provider configuration or add instances"
  long_term: "Implement capacity monitoring and planning process"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
