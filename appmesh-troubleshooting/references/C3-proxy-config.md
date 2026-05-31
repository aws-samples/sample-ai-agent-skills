---
title: "C3 — Envoy Proxy Configuration Issues"
description: "Diagnose Envoy proxy configuration delivery and application problems"
status: active
severity: MEDIUM
triggers:
  - "envoy config"
  - "config dump"
  - "proxy misconfigured"
  - "wrong route in envoy"
  - "stale config"
owner: devops-agent
objective: "Identify and resolve Envoy proxy configuration issues"
context: "App Mesh pushes configuration to Envoy via the Envoy Management Service (EMS). Configuration issues manifest as stale routes, missing clusters, or incorrect listener settings in Envoy."
---

## Phase 1 — Triage

MUST:
- Get Envoy config dump: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/config_dump | python3 -m json.tool > /tmp/envoy-config.json`
- Check Envoy clusters: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/clusters`
- Check Envoy listeners: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/listeners`
- Verify config version: check the version_info in config dump
- Compare Envoy config with App Mesh resource definitions

SHOULD:
- Check for stale configuration (old version_info)
- Verify all expected clusters are present
- Check listener filter chains match expected routing

MAY:
- Enable Envoy debug logging: `kubectl exec <pod-name> -c envoy -- curl -s -X POST http://localhost:9901/logging?level=debug`
- Check Envoy xDS connection status

## Phase 2 — Remediate

MUST:
- Verify App Mesh resources match expected configuration
- Restart Envoy if configuration is stale (delete pod to trigger re-injection)
- Fix App Mesh resource definitions if they don't match desired routing

SHOULD:
- Monitor config version to detect stale configurations
- Verify EMS connectivity for config delivery
- Check for App Mesh API throttling that may delay config delivery

MAY:
- Use Envoy admin API to force config refresh
- Set up config dump comparison tooling

## Common Issues

- symptoms: "Envoy has old routes that don't match current App Mesh config"
  diagnosis: "Configuration delivery from App Mesh to Envoy is delayed or failed."
  resolution: "Check EMS connectivity. Restart the pod to force fresh config delivery."

- symptoms: "Expected cluster missing from Envoy"
  diagnosis: "Backend virtual service not configured on the virtual node."
  resolution: "Add the missing backend to the virtual node configuration."

- symptoms: "Envoy listener on wrong port"
  diagnosis: "Virtual node listener port doesn't match application port."
  resolution: "Update virtual node listener to match the application's listening port."

## Output Format

```yaml
root_cause: "proxy_config_issue — <specific_cause>"
evidence:
  - type: envoy_config
    content: "<config dump summary>"
  - type: appmesh_config
    content: "<App Mesh resource definitions>"
  - type: config_version
    content: "<Envoy config version info>"
severity: MEDIUM
mitigation:
  immediate: "Fix App Mesh resources or restart pod for fresh config"
  long_term: "Implement config drift detection between App Mesh and Envoy"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify App Mesh resources match config | GREEN | Read-only verification |
| Restart Envoy (delete pod) | YELLOW | Causes brief service disruption |
| Fix App Mesh resource definitions | YELLOW | Changes routing configuration |
| Monitor config version | GREEN | Observability |
| Enable Envoy debug logging | YELLOW | Increases log volume |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Envoy config dump**: Full proxy configuration — INTERNAL
- **App Mesh resource definitions**: Service mesh topology — INTERNAL
- **Config version info**: xDS delivery status — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER manually edit Envoy configuration files — use App Mesh API
- NEVER force config refresh on all pods simultaneously in production

## Phase 3 — Rollback

1. Revert App Mesh resource definitions to previous state
2. Delete and recreate affected pods to force fresh config delivery
3. Verify Envoy config dump matches expected configuration
4. Check all clusters and listeners are present
5. Monitor service communication for errors after config refresh

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
