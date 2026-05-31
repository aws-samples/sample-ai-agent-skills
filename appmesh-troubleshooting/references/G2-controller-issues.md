---
title: "G2 — App Mesh Controller Issues"
description: "Diagnose App Mesh controller problems on EKS"
status: active
severity: HIGH
triggers:
  - "controller crash"
  - "controller error"
  - "controller not reconciling"
  - "webhook error"
  - "controller restart"
owner: devops-agent
objective: "Identify and resolve App Mesh controller issues affecting mesh operations on EKS"
context: "The App Mesh controller translates Kubernetes CRDs into App Mesh API calls. Controller issues affect all mesh operations including resource creation, updates, and sidecar injection."
---

## Phase 1 — Triage

MUST:
- Check controller pod status: `kubectl get pods -n appmesh-system -l app.kubernetes.io/name=appmesh-controller`
- Check controller logs: `kubectl logs -n appmesh-system -l app.kubernetes.io/name=appmesh-controller --tail=200`
- Check controller events: `kubectl describe pod -n appmesh-system -l app.kubernetes.io/name=appmesh-controller`
- Verify controller image version: `kubectl get deployment -n appmesh-system appmesh-controller -o jsonpath='{.spec.template.spec.containers[0].image}'`
- Check IRSA configuration: `kubectl describe sa -n appmesh-system appmesh-controller`

SHOULD:
- Check controller resource usage (CPU, memory)
- Verify controller can reach AWS APIs
- Check for controller leader election issues (if multiple replicas)

MAY:
- Check controller metrics endpoint
- Review controller Helm release: `helm list -n appmesh-system`

## Phase 2 — Remediate

MUST:
- Fix IRSA if controller lacks AWS permissions
- Restart controller if stuck: `kubectl rollout restart deployment -n appmesh-system appmesh-controller`
- Upgrade controller if version is incompatible

SHOULD:
- Increase controller resource limits if OOM killed
- Verify controller Helm chart values are correct
- Check for known issues in the controller version

MAY:
- Enable controller debug logging
- Set up controller health monitoring and alerting

## Common Issues

- symptoms: "Controller in CrashLoopBackOff"
  diagnosis: "Controller cannot start — missing IRSA, incompatible version, or resource limits."
  resolution: "Check controller logs for startup errors. Fix IRSA or upgrade controller."

- symptoms: "Controller running but not reconciling resources"
  diagnosis: "Controller stuck or leader election failed."
  resolution: "Restart controller deployment. Check for multiple controller instances."

- symptoms: "Webhook timeout errors on pod creation"
  diagnosis: "Controller webhook is slow or unresponsive."
  resolution: "Check controller health and resource usage. Restart if needed."

## Output Format

```yaml
root_cause: "controller_issue — <specific_cause>"
evidence:
  - type: controller_status
    content: "<pod status and events>"
  - type: controller_logs
    content: "<error messages>"
  - type: irsa
    content: "<IAM role configuration>"
severity: HIGH
mitigation:
  immediate: "Fix controller configuration and restart"
  long_term: "Implement controller monitoring and automated recovery"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix IRSA permissions | YELLOW | IAM configuration change |
| Restart controller | YELLOW | Temporary loss of mesh reconciliation |
| Upgrade controller version | YELLOW | Component version change |
| Increase controller resource limits | GREEN | Resource allocation |
| Enable controller debug logging | YELLOW | Increases log volume |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Controller logs**: Error messages and reconciliation state — INTERNAL
- **IRSA configuration**: IAM role details — INTERNAL
- **Helm release**: Controller deployment configuration — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER delete the controller deployment without understanding impact on mesh operations
- NEVER run multiple controller instances without leader election configured

## Phase 3 — Rollback

1. Rollback controller: `kubectl rollout undo deployment -n appmesh-system appmesh-controller`
2. Revert IRSA changes if IAM was modified
3. Disable debug logging if it was enabled
4. Verify controller is reconciling mesh resources
5. Check webhook is responding to pod creation events

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
