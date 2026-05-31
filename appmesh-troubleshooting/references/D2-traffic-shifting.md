---
title: "D2 — App Mesh Traffic Shifting Issues"
description: "Diagnose traffic shifting, canary, and blue/green deployment problems"
status: active
severity: MEDIUM
triggers:
  - "traffic shifting"
  - "canary deployment"
  - "blue green"
  - "weighted routing"
  - "traffic split"
owner: devops-agent
objective: "Identify and resolve traffic shifting configuration issues"
context: "Traffic shifting uses weighted routes to distribute traffic between virtual nodes. Used for canary deployments, blue/green deployments, and gradual rollouts. Weights determine traffic distribution percentage."
---

## Phase 1 — Triage

MUST:
- Check route weights: `aws appmesh describe-route --mesh-name <mesh-name> --virtual-router-name <router-name> --route-name <route-name> --query 'route.spec.httpRoute.action.weightedTargets'`
- Verify both target virtual nodes exist and are healthy
- Check Envoy weighted cluster configuration: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/clusters`
- Verify traffic distribution matches expected weights

SHOULD:
- Check health of both canary and stable virtual nodes
- Verify service discovery resolves for both targets
- Monitor error rates during traffic shift

MAY:
- Check Envoy stats for per-cluster request counts
- Verify no circuit breaker is tripping on the canary

## Phase 2 — Remediate

MUST:
- Ensure both target virtual nodes are healthy before shifting traffic
- Set weights correctly (e.g., 90/10 for 10% canary)
- Roll back weights if canary shows errors

SHOULD:
- Shift traffic gradually (e.g., 5% → 10% → 25% → 50% → 100%)
- Monitor error rates and latency during each shift
- Set up automated rollback based on error thresholds

MAY:
- Use header-based routing for internal canary testing before weighted shift
- Implement progressive delivery tooling (Flagger, Argo Rollouts)

## Common Issues

- symptoms: "All traffic goes to one target despite 50/50 weights"
  diagnosis: "One target virtual node is unhealthy, so Envoy routes all traffic to the healthy one."
  resolution: "Fix the unhealthy virtual node. Check health checks and service discovery."

- symptoms: "Traffic distribution doesn't match configured weights"
  diagnosis: "Envoy config hasn't been updated yet or weights are per-connection not per-request."
  resolution: "Wait for config propagation. Verify with Envoy stats over a sufficient sample size."

- symptoms: "Canary receiving errors but traffic not rolling back"
  diagnosis: "No automatic rollback configured — App Mesh doesn't auto-rollback."
  resolution: "Manually update weights to shift traffic back. Implement automated rollback."

## Output Format

```yaml
root_cause: "traffic_shifting_issue — <specific_cause>"
evidence:
  - type: route_weights
    content: "<configured weights>"
  - type: target_health
    content: "<health status of both targets>"
  - type: traffic_distribution
    content: "<actual traffic distribution>"
severity: MEDIUM
mitigation:
  immediate: "Fix target health or adjust weights"
  long_term: "Implement progressive delivery with automated rollback"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure both target nodes are healthy | GREEN | Health verification |
| Set weights correctly (e.g., 90/10) | YELLOW | Changes traffic distribution |
| Roll back weights if canary shows errors | RED | Emergency traffic shift |
| Shift traffic gradually | YELLOW | Progressive traffic change |
| Monitor error rates during shift | GREEN | Observability |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Route weights**: Traffic distribution percentages — INTERNAL
- **Target health**: Virtual node health status — INTERNAL
- **Traffic metrics**: Per-cluster request counts — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER shift 100% traffic to an untested canary in one step
- NEVER modify traffic weights during an active incident without approval

## Phase 3 — Rollback

1. Immediately shift all traffic back to stable: set weights to 100/0 for stable/canary
2. Verify Envoy weighted cluster configuration reflects the rollback
3. Monitor error rates to confirm rollback resolved the issue
4. Investigate canary failures before attempting another shift
5. Document the rollback reason and timeline

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
