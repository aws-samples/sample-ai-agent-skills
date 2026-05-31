---
title: "B3 — App Mesh Virtual Router Issues"
description: "Diagnose virtual router and route configuration problems"
status: active
severity: MEDIUM
triggers:
  - "virtual router"
  - "route not matching"
  - "router error"
  - "listener mismatch"
owner: devops-agent
objective: "Identify and resolve virtual router configuration issues"
context: "Virtual routers define listeners and contain routes that direct traffic to virtual nodes. Router listener protocol must match the traffic type. Routes define matching rules and target weights."
---

## Phase 1 — Triage

MUST:
- Describe virtual router: `aws appmesh describe-virtual-router --mesh-name <mesh-name> --virtual-router-name <router-name>`
- List routes: `aws appmesh list-routes --mesh-name <mesh-name> --virtual-router-name <router-name>`
- Check each route: `aws appmesh describe-route --mesh-name <mesh-name> --virtual-router-name <router-name> --route-name <route-name>`
- Verify listener protocol matches traffic (http, http2, grpc, tcp)

SHOULD:
- Check route priority ordering
- Verify route match conditions (path, headers, method)
- Check target virtual node weights

MAY:
- Check Envoy route configuration: `curl -s http://localhost:9901/config_dump`
- Verify all target virtual nodes exist and are healthy

## Phase 2 — Remediate

MUST:
- Set router listener protocol to match application traffic
- Configure routes with correct match conditions
- Ensure target virtual nodes exist and are healthy

SHOULD:
- Order routes from most specific to least specific
- Use appropriate route types (http, tcp, grpc) matching the protocol
- Set retry policies and timeouts on routes

MAY:
- Add header-based routing for advanced traffic management
- Implement route-level access logging

## Common Issues

- symptoms: "Traffic not matching any route"
  diagnosis: "Route match conditions don't match the actual request."
  resolution: "Review route match rules (path prefix, headers) and fix to match actual traffic."

- symptoms: "Wrong route matched"
  diagnosis: "Route priority ordering incorrect — less specific route matched first."
  resolution: "Reorder routes so more specific matches have higher priority."

- symptoms: "Router listener protocol mismatch"
  diagnosis: "Router listener set to HTTP but application uses gRPC."
  resolution: "Update router listener protocol to match the application protocol."

## Output Format

```yaml
root_cause: "virtual_router_issue — <specific_cause>"
evidence:
  - type: router_config
    content: "<listener and route configuration>"
  - type: route_details
    content: "<match conditions and targets>"
  - type: envoy_routes
    content: "<Envoy route table>"
severity: MEDIUM
mitigation:
  immediate: "Fix router listener or route configuration"
  long_term: "Document routing rules and implement route testing"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Set router listener protocol | YELLOW | Affects traffic handling |
| Configure routes with correct match conditions | YELLOW | Changes traffic routing |
| Ensure target virtual nodes are healthy | GREEN | Health verification |
| Order routes by specificity | YELLOW | Changes route evaluation order |
| Set retry policies and timeouts | GREEN | Resilience configuration |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Router configuration**: Listener protocol and port — INTERNAL
- **Route rules**: Match conditions, targets, weights — INTERNAL
- **Envoy route tables**: Proxy routing configuration — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER reorder routes in production without understanding current traffic patterns
- NEVER delete routes that are actively serving traffic

## Phase 3 — Rollback

1. Revert route configuration to previous match conditions and targets
2. Restore previous route priority ordering
3. Revert router listener protocol if changed
4. Verify Envoy config reflects the rollback
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
