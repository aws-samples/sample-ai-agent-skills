---
title: "B2 — App Mesh Virtual Service Issues"
description: "Diagnose virtual service routing and provider configuration problems"
status: active
severity: HIGH
triggers:
  - "virtual service"
  - "service not routing"
  - "provider not configured"
  - "service unreachable"
owner: devops-agent
objective: "Identify and resolve virtual service configuration issues"
context: "Virtual services are the abstraction layer clients connect to. They must have a provider (virtual router or virtual node) configured. Without a provider, traffic is not routed."
---

## Phase 1 — Triage

MUST:
- Describe virtual service: `aws appmesh describe-virtual-service --mesh-name <mesh-name> --virtual-service-name <service-name>`
- Check provider configuration: look at provider in the service spec
- If provider is virtual router, check router: `aws appmesh describe-virtual-router --mesh-name <mesh-name> --virtual-router-name <router-name>`
- If provider is virtual node, check node: `aws appmesh describe-virtual-node --mesh-name <mesh-name> --virtual-node-name <node-name>`
- Verify the virtual service name matches what clients use for DNS resolution

SHOULD:
- Check Envoy config for the virtual service: `curl -s http://localhost:9901/config_dump | python3 -c "import sys,json; print(json.dumps([c for c in json.load(sys.stdin)['configs'] if 'route' in str(c).lower()][:2], indent=2))"`
- Verify the provider resource exists and is healthy

MAY:
- Check if multiple virtual services conflict on the same hostname
- Verify DNS resolution matches the virtual service name

## Phase 2 — Remediate

MUST:
- Configure a provider (virtual router or virtual node) on the virtual service
- Ensure the provider resource exists and is correctly configured
- Match the virtual service name with the DNS name clients use

SHOULD:
- Use virtual router as provider for advanced routing (weighted, path-based)
- Use virtual node as provider for simple direct routing
- Document the routing topology

MAY:
- Set up monitoring for virtual service health
- Implement service mesh visualization

## Common Issues

- symptoms: "Clients get connection refused when calling the service"
  diagnosis: "Virtual service has no provider configured."
  resolution: "Add a virtual router or virtual node as the provider."

- symptoms: "Traffic goes to wrong destination"
  diagnosis: "Virtual service name doesn't match the DNS name clients resolve."
  resolution: "Ensure virtual service name matches the service's DNS hostname."

- symptoms: "Virtual service exists but Envoy doesn't route to it"
  diagnosis: "Calling virtual node doesn't list this virtual service as a backend."
  resolution: "Add the virtual service as a backend on the calling virtual node."

## Output Format

```yaml
root_cause: "virtual_service_issue — <specific_cause>"
evidence:
  - type: service_config
    content: "<provider configuration>"
  - type: provider_status
    content: "<virtual router or node details>"
  - type: envoy_config
    content: "<Envoy route configuration>"
severity: HIGH
mitigation:
  immediate: "Configure provider and verify routing"
  long_term: "Document service mesh topology and routing rules"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Configure a provider (router or node) | YELLOW | Changes traffic routing |
| Ensure provider resource exists | GREEN | Verification only |
| Match virtual service name with DNS | YELLOW | Affects service resolution |
| Use virtual router for advanced routing | YELLOW | Routing topology change |
| Set up monitoring | GREEN | Observability setup |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Virtual service config**: Provider and routing topology — INTERNAL
- **DNS configuration**: Service name resolution — INTERNAL
- **Envoy route config**: Proxy routing tables — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER change a virtual service provider in production without testing
- NEVER delete a virtual service that is referenced as a backend by other nodes

## Phase 3 — Rollback

1. Revert virtual service provider to previous configuration
2. Verify Envoy config reflects the rollback
3. Check that all calling virtual nodes can still reach this service
4. Monitor traffic flow and error rates
5. Verify DNS resolution matches the virtual service name

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
