---
title: "B1 — App Mesh Virtual Node Issues"
description: "Diagnose virtual node configuration and connectivity problems"
status: active
severity: HIGH
triggers:
  - "virtual node"
  - "node not reachable"
  - "service discovery"
  - "backend not found"
  - "node health"
owner: devops-agent
objective: "Identify and resolve virtual node configuration issues affecting service connectivity"
context: "Virtual nodes represent actual compute resources (ECS tasks, EKS pods, EC2 instances). They require service discovery, listener configuration, and backend definitions. Misconfigured virtual nodes cause routing failures."
---

## Phase 1 — Triage

MUST:
- Describe virtual node: `aws appmesh describe-virtual-node --mesh-name <mesh-name> --virtual-node-name <node-name>`
- Check service discovery config: look at serviceDiscovery in the node spec
- Check backends: `aws appmesh describe-virtual-node --mesh-name <mesh-name> --virtual-node-name <node-name> --query 'virtualNode.spec.backends'`
- Check listener configuration: look at listeners in the node spec
- Verify health check configuration: look at healthCheck in listener spec

SHOULD:
- Verify DNS or Cloud Map service discovery resolves correctly
- Check Envoy proxy clusters for the node: `curl -s http://localhost:9901/clusters`
- Verify backend virtual services exist and have providers

MAY:
- Check Envoy stats for connection errors: `curl -s http://localhost:9901/stats | grep upstream_cx`
- Verify TLS configuration on listeners and backends

## Phase 2 — Remediate

MUST:
- Configure service discovery (DNS hostname or Cloud Map namespace/service)
- Add all required backends (virtual services this node needs to call)
- Configure listeners with correct port and protocol

SHOULD:
- Set up health checks appropriate for the service
- Configure TLS on listeners for encrypted communication
- Match listener protocol with actual application protocol

MAY:
- Add connection pool settings for high-traffic nodes
- Configure outlier detection for automatic unhealthy host ejection

## Common Issues

- symptoms: "Virtual node not reachable from other nodes"
  diagnosis: "Service discovery not configured or DNS not resolving."
  resolution: "Configure DNS or Cloud Map service discovery. Verify DNS resolution."

- symptoms: "Node cannot call backend services"
  diagnosis: "Backend virtual services not listed in the node's backends."
  resolution: "Add the target virtual services as backends on the virtual node."

- symptoms: "Health check failing on virtual node"
  diagnosis: "Health check path, port, or protocol doesn't match the application."
  resolution: "Update health check configuration to match the application's health endpoint."

## Output Format

```yaml
root_cause: "virtual_node_issue — <specific_cause>"
evidence:
  - type: node_config
    content: "<service discovery, listeners, backends>"
  - type: envoy_clusters
    content: "<Envoy cluster health status>"
  - type: health_check
    content: "<health check configuration and results>"
severity: HIGH
mitigation:
  immediate: "Fix service discovery, backends, or health check configuration"
  long_term: "Standardize virtual node configurations across services"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Configure service discovery | YELLOW | Affects how services are found |
| Add required backends | YELLOW | Changes traffic routing |
| Configure listeners with correct port/protocol | YELLOW | Affects traffic reception |
| Set up health checks | GREEN | Monitoring configuration |
| Configure TLS on listeners | YELLOW | Encryption configuration change |
| Add connection pool settings | YELLOW | Affects traffic handling capacity |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Service discovery config**: DNS hostnames, Cloud Map namespaces — INTERNAL
- **Backend configuration**: Service-to-service communication topology — INTERNAL
- **Envoy cluster data**: Proxy health and connection stats — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER remove backends from a virtual node without verifying no traffic flows to them
- NEVER change listener ports on production virtual nodes without coordinated deployment

## Phase 3 — Rollback

1. Revert virtual node configuration to previous state: `aws appmesh update-virtual-node --mesh-name <mesh> --virtual-node-name <node> --spec <previous-spec>`
2. Verify Envoy proxy receives updated configuration
3. Check service discovery resolution is working
4. Verify health checks are passing
5. Monitor service-to-service communication for errors

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
