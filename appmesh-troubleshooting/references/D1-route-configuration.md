---
title: "D1 — App Mesh Route Configuration Issues"
description: "Diagnose route matching and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "route not matching"
  - "wrong destination"
  - "route configuration"
  - "path matching"
  - "header matching"
owner: devops-agent
objective: "Identify and resolve route configuration issues affecting traffic routing"
context: "Routes define how traffic is matched and directed to virtual nodes. Routes support path prefix, exact path, header, and method matching. Route priority determines evaluation order."
---

## Phase 1 — Triage

MUST:
- List routes on the router: `aws appmesh list-routes --mesh-name <mesh-name> --virtual-router-name <router-name>`
- Describe each route: `aws appmesh describe-route --mesh-name <mesh-name> --virtual-router-name <router-name> --route-name <route-name>`
- Check route match conditions: path prefix, exact path, headers, method
- Check route priority ordering
- Verify target virtual nodes exist and are healthy

SHOULD:
- Check Envoy route table: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/config_dump`
- Verify route type matches traffic protocol (http, grpc, tcp)
- Check retry policy and timeout settings

MAY:
- Test route matching with specific requests
- Check Envoy stats for route-level metrics

## Phase 2 — Remediate

MUST:
- Fix route match conditions to correctly match intended traffic
- Set route priorities so specific routes are evaluated before general ones
- Ensure target virtual nodes are healthy

SHOULD:
- Configure retry policies for resilience
- Set appropriate timeouts for backend response times
- Use header-based routing for advanced traffic management

MAY:
- Add route-level access logging for debugging
- Implement route testing in CI/CD

## Common Issues

- symptoms: "All traffic goes to one route, ignoring others"
  diagnosis: "A catch-all route (prefix /) has higher priority than specific routes."
  resolution: "Set specific routes with higher priority than the catch-all route."

- symptoms: "gRPC traffic not matching HTTP routes"
  diagnosis: "Route type is HTTP but traffic is gRPC."
  resolution: "Use gRPC route type with service name and method matching."

- symptoms: "Route matches but returns 503"
  diagnosis: "Target virtual node is unhealthy or has no healthy endpoints."
  resolution: "Check target virtual node health and service discovery."

## Output Format

```yaml
root_cause: "route_configuration_issue — <specific_cause>"
evidence:
  - type: route_config
    content: "<match conditions, targets, priorities>"
  - type: envoy_routes
    content: "<Envoy route table>"
  - type: target_health
    content: "<target virtual node health>"
severity: MEDIUM
mitigation:
  immediate: "Fix route match conditions or priority ordering"
  long_term: "Document routing rules and implement route testing"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix route match conditions | YELLOW | Changes traffic routing |
| Set route priorities | YELLOW | Changes route evaluation order |
| Ensure target virtual nodes are healthy | GREEN | Health verification |
| Configure retry policies | GREEN | Resilience improvement |
| Set appropriate timeouts | GREEN | Timeout configuration |
| Add header-based routing | YELLOW | New routing rules |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Route configuration**: Match conditions, targets, priorities — INTERNAL
- **Envoy route tables**: Proxy routing state — INTERNAL
- **Target health**: Virtual node health status — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER modify route match conditions in production without understanding current traffic
- NEVER delete the catch-all route without ensuring specific routes cover all traffic

## Phase 3 — Rollback

1. Revert route match conditions to previous configuration
2. Restore previous route priority ordering
3. Revert retry policies and timeouts if changed
4. Verify Envoy route table reflects the rollback
5. Monitor traffic routing and error rates for 15 minutes

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
