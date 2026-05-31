---
title: "Z1 — General App Mesh Troubleshooting (Catch-All)"
description: "Fallback SOP for App Mesh issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown App Mesh issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all meshes: `aws appmesh list-meshes`
- Describe mesh: `aws appmesh describe-mesh --mesh-name <mesh-name>`
- List virtual nodes: `aws appmesh list-virtual-nodes --mesh-name <mesh-name>`
- List virtual services: `aws appmesh list-virtual-services --mesh-name <mesh-name>`
- List virtual routers: `aws appmesh list-virtual-routers --mesh-name <mesh-name>`
- Check Envoy sidecar presence: `kubectl get pods -n <namespace> -o jsonpath='{.items[*].spec.containers[*].name}'`

SHOULD:
- Check Envoy health: `kubectl exec <pod-name> -c envoy -- curl -s http://localhost:9901/ready`
- Check App Mesh controller (EKS): `kubectl get pods -n appmesh-system`
- Check CloudTrail for recent changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appmesh.amazonaws.com --max-results 20`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Mesh creation/configuration → Mesh (A1-A2)
- Virtual node/service/router issues → Virtual (B1-B3)
- Envoy proxy problems → Envoy (C1-C3)
- Route matching/traffic issues → Routing (D1-D2)
- TLS/mTLS failures → TLS (E1-E2)
- Logging/tracing issues → Observability (F1-F2)
- EKS/controller problems → EKS (G1-G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check Envoy config dump for configuration state
- Review Envoy stats for error patterns
- Verify all mesh resources are correctly linked
- Check IAM permissions for all mesh operations

SHOULD:
- Check AWS Health Dashboard for App Mesh service events
- Compare with known-good mesh configuration
- Review recent changes in CloudTrail

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<mesh|virtual|envoy|routing|tls|observability|eks|unknown>"
investigation_path: "mesh → virtual resources → envoy → <domain_classification>"
evidence:
  - type: mesh_config
    content: "<mesh configuration summary>"
  - type: envoy_status
    content: "<Envoy health and config>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| List all meshes | GREEN | Read-only check |
| Describe mesh | GREEN | Read-only check |
| List virtual nodes/services/routers | GREEN | Read-only check |
| Check Envoy sidecar presence | GREEN | Read-only check |
| Check Envoy health | GREEN | Read-only check |
| Check App Mesh controller | GREEN | Read-only check |
| Check CloudTrail events | GREEN | Read-only audit |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Mesh topology**: Full service mesh configuration — INTERNAL
- **Envoy status**: Proxy health and configuration — INTERNAL
- **CloudTrail events**: API call history — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER make changes to production mesh resources during initial triage
- NEVER restart all Envoy proxies simultaneously

## Phase 3 — Rollback

1. Revert any configuration changes made during investigation
2. Restart affected pods if Envoy configuration was modified
3. Verify all mesh resources are in expected state
4. Check Envoy health across all services
5. Document findings and escalate if root cause is unclassified

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
