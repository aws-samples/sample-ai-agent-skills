---
title: "F1 — Cluster Management Service"
description: "Diagnose EKS Anywhere cluster management service and controller issues"
status: active
severity: HIGH
triggers: ["eksa controller error", "cluster management failed", "CAPI controller crash", "eksa-system pods failing"]
owner: devops-agent
objective: "Resolve cluster management service issues including CAPI controllers and EKS Anywhere controllers"
context: "The management cluster runs CAPI and EKS Anywhere controllers that manage workload cluster lifecycle. Controller failures affect all managed workload clusters."
---

## Phase 1 — Triage

MUST:
- Check EKS Anywhere controller: `kubectl get pods -n eksa-system --kubeconfig <mgmt-kubeconfig>`
- Check CAPI controllers: `kubectl get pods -n capi-system --kubeconfig <mgmt-kubeconfig>`
- Check provider controllers: `kubectl get pods -n capv-system --kubeconfig <mgmt-kubeconfig>` (or capt-system for Tinkerbell)

SHOULD:
- Review EKS Anywhere controller logs: `kubectl logs -n eksa-system deployment/eksa-controller-manager --tail=200`
- Review CAPI controller logs: `kubectl logs -n capi-system deployment/capi-controller-manager --tail=200`
- Check controller resource usage: `kubectl top pods -n eksa-system --kubeconfig <mgmt-kubeconfig>`

MAY:
- Check webhook configurations: `kubectl get validatingwebhookconfigurations --kubeconfig <mgmt-kubeconfig>`
- Review controller leader election status

## Phase 2 — Remediate

MUST:
- Fix controller crashes (OOMKilled, CrashLoopBackOff) by checking resource limits and logs
- Resolve webhook certificate expiry blocking API operations
- Ensure management cluster has sufficient resources for controllers

SHOULD:
- Restart specific failing controllers: `kubectl rollout restart deployment/<controller> -n <namespace>`
- Verify cert-manager is healthy (provides certificates for webhooks)

MAY:
- Increase controller resource limits if OOMKilled
- Check for conflicting CRD versions after upgrades

## Safety Ratings

```
safety_ratings:
  - "kubectl get pods controllers: GREEN — read-only status check"
  - "kubectl logs controllers: GREEN — read-only log retrieval"
  - "kubectl rollout restart: YELLOW — briefly disrupts controller operations"
  - "kubectl top pods: GREEN — read-only resource usage check"
```

## Escalation Conditions

- All CAPI controllers are down preventing any workload cluster management
- Management cluster itself is unhealthy (etcd issues, API server errors)
- Controller crashes caused by corrupted CRD state requiring manual intervention
- Webhook failures blocking all cluster operations

## Data Sensitivity

- `kubectl get pods controllers`: LOW — returns pod status
- `kubectl logs controllers`: MEDIUM — may contain cluster configuration and API calls
- `kubectl get validatingwebhookconfigurations`: LOW — returns webhook configuration
- `controller resource usage`: LOW — returns CPU/memory metrics

## Prohibited Actions

- NEVER delete CAPI CRDs — this orphans all managed cluster resources
- NEVER delete the management cluster without first migrating or deleting workload clusters
- NEVER restart all controllers simultaneously — restart one at a time
- NEVER modify controller deployments directly — use eksctl anywhere for upgrades

## Phase 3 — Rollback

1. If a controller was restarted and is worse, check previous ReplicaSet: `kubectl rollout undo deployment/<controller> -n <namespace>`
2. If CRDs were modified, restore from backup or re-install the correct version
3. If webhook certificates expired, restart cert-manager and wait for renewal
4. Verify all controllers are running: `kubectl get pods -n eksa-system -n capi-system -n capv-system`
5. Confirm workload cluster management operations work

## Common Issues

- symptoms: "eksa-controller-manager CrashLoopBackOff — OOMKilled"
  diagnosis: "Controller memory limit too low for the number of managed clusters."
  resolution: "Increase memory limit in the controller deployment."

## Output Format

```yaml
root_cause: "cluster_management — <specific_cause>"
evidence:
  - type: controller_error
    content: "<error details from controller logs>"
severity: HIGH
mitigation:
  immediate: "Restore controller health"
  long_term: "Monitor controller resources and set up alerts"
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
  - command: "describe-cluster"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get configmap aws-auth"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get pods"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest system:masters group for troubleshooting"
  - "NEVER suggest disabling RBAC or Pod Security Standards"
  - "NEVER suggest running pods as root to fix permissions"
