---
title: "G1 — App Mesh EKS Integration Issues"
description: "Diagnose App Mesh integration problems with Amazon EKS"
status: active
severity: HIGH
triggers:
  - "EKS integration"
  - "mesh on EKS"
  - "kubernetes mesh"
  - "CRD error"
  - "mesh resource"
owner: devops-agent
objective: "Identify and resolve App Mesh integration issues on EKS clusters"
context: "App Mesh on EKS uses the App Mesh controller to translate Kubernetes CRDs into App Mesh API calls. Integration requires the controller, CRDs, and proper IRSA configuration."
---

## Phase 1 — Triage

MUST:
- Check App Mesh controller: `kubectl get pods -n appmesh-system`
- Check CRDs installed: `kubectl get crds | grep appmesh`
- Check mesh CRs: `kubectl get meshes,virtualnodes,virtualservices,virtualrouters -A`
- Check controller logs: `kubectl logs -n appmesh-system -l app.kubernetes.io/name=appmesh-controller --tail=100`
- Verify IRSA for controller: `kubectl get sa -n appmesh-system appmesh-controller -o yaml`

SHOULD:
- Check controller version: `kubectl get deployment -n appmesh-system appmesh-controller -o jsonpath='{.spec.template.spec.containers[0].image}'`
- Verify webhook configuration: `kubectl get mutatingwebhookconfigurations | grep appmesh`
- Check for CR status conditions: `kubectl get virtualnode <name> -n <namespace> -o yaml`

MAY:
- Check controller RBAC: `kubectl get clusterrole appmesh-controller -o yaml`
- Verify controller can reach App Mesh API

## Phase 2 — Remediate

MUST:
- Install or upgrade App Mesh controller via Helm
- Install required CRDs
- Configure IRSA with App Mesh permissions for the controller service account

SHOULD:
- Use the latest compatible controller version
- Verify webhook injection is working
- Test with a simple mesh configuration first

MAY:
- Enable controller debug logging for troubleshooting
- Set up controller health monitoring

## Common Issues

- symptoms: "Kubernetes mesh resources created but not reflected in App Mesh"
  diagnosis: "App Mesh controller not running or lacks IAM permissions."
  resolution: "Check controller pod status and IRSA configuration."

- symptoms: "Controller running but CRs show error status"
  diagnosis: "Controller cannot create App Mesh resources — IAM or API errors."
  resolution: "Check controller logs for specific errors. Verify IRSA permissions."

- symptoms: "Sidecar injection not working on new pods"
  diagnosis: "Mutating webhook not configured or namespace not annotated."
  resolution: "Verify webhook configuration and add namespace annotation."

## Output Format

```yaml
root_cause: "eks_integration_issue — <specific_cause>"
evidence:
  - type: controller_status
    content: "<controller pod status and logs>"
  - type: crds
    content: "<installed CRDs and CR statuses>"
  - type: irsa
    content: "<service account IAM role>"
severity: HIGH
mitigation:
  immediate: "Fix controller, CRDs, or IRSA configuration"
  long_term: "Automate App Mesh EKS setup with Helm and GitOps"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Install/upgrade App Mesh controller via Helm | YELLOW | Cluster-wide component change |
| Install required CRDs | YELLOW | Cluster-wide schema change |
| Configure IRSA with App Mesh permissions | YELLOW | IAM role configuration |
| Verify webhook injection | GREEN | Verification only |
| Test with simple mesh configuration | GREEN | Non-production testing |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Controller logs**: Reconciliation events and errors — INTERNAL
- **IRSA configuration**: IAM role and service account mapping — INTERNAL
- **CRD definitions**: Mesh resource schemas — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER upgrade the App Mesh controller in production without testing in staging
- NEVER delete CRDs as this will delete all mesh custom resources

## Phase 3 — Rollback

1. Rollback controller to previous version: `helm rollback appmesh-controller -n appmesh-system`
2. Revert IRSA configuration if changed
3. Verify all mesh custom resources are still present and healthy
4. Check controller logs for reconciliation errors
5. Monitor sidecar injection on new pods

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
