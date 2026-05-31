---
title: "G3 — HPA Issues"
description: "Diagnose and resolve Horizontal Pod Autoscaler scaling problems"
status: active
severity: MEDIUM
triggers:
  - "HPA"
  - "Horizontal Pod Autoscaler"
  - "not scaling"
  - "target CPU"
  - "metrics"
owner: devops-agent
objective: "Identify why HPA is not scaling pods and fix the metrics or configuration"
context: "HPA scales pod replicas based on CPU, memory, or custom metrics. It requires metrics-server for CPU/memory metrics. HPA checks metrics every 15 seconds (default) and scales based on the ratio of current to target metric value. Common issues: metrics-server not installed, no resource requests set, or metric unavailable."
---

## Phase 1 — Triage

MUST:
- Check HPA status: `kubectl get hpa -n <namespace>`
- Describe the HPA for events and conditions: `kubectl describe hpa <name> -n <namespace>`
- Check if metrics-server is running: `kubectl get pods -n kube-system -l k8s-app=metrics-server`
- Verify pod resource requests are set (required for CPU/memory HPA): `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].resources.requests}'`
- Test metrics availability: `kubectl top pods -n <namespace>`

SHOULD:
- Check the HPA's current and target metric values
- Verify the HPA min/max replica settings
- Check if the deployment has enough quota to scale
- Look for "unable to fetch metrics" or "missing request" in HPA events

MAY:
- Check custom metrics adapter if using custom/external metrics
- Verify Prometheus adapter configuration for custom metrics
- Check HPA behavior configuration (scaleUp/scaleDown policies)

## Phase 2 — Remediate

MUST:
- If metrics-server not installed: install it: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`
- If resource requests missing: add CPU/memory requests to the pod spec
- If HPA target too high/low: adjust the target utilization percentage
- If metrics unavailable: fix the metrics pipeline (metrics-server, Prometheus adapter)

SHOULD:
- Set both min and max replicas appropriately
- Use `behavior` field to control scale-up/scale-down speed
- Set resource requests based on actual usage (use VPA recommendations)

MAY:
- Use multiple metrics (CPU + custom) for more accurate scaling
- Configure scale-down stabilization window to prevent flapping
- Use KEDA for event-driven autoscaling

## Common Issues

- symptoms: "HPA shows '<unknown>/80%' for CPU target"
  diagnosis: "metrics-server is not installed or not working."
  resolution: "Install metrics-server. Check it can reach kubelets (port 10250)."

- symptoms: "HPA shows 'missing request for cpu' in events"
  diagnosis: "Pod containers don't have CPU resource requests set."
  resolution: "Add CPU requests to all containers in the pod spec."

- symptoms: "HPA scales to max but pods are still overloaded"
  diagnosis: "Max replicas too low, or the bottleneck is not CPU/memory."
  resolution: "Increase maxReplicas. Check if the bottleneck is external (database, API)."

- symptoms: "HPA keeps scaling up and down rapidly (flapping)"
  diagnosis: "Target utilization is too close to actual usage, causing oscillation."
  resolution: "Add scale-down stabilization window. Adjust target utilization with more headroom."

## Output Format

```yaml
root_cause: "HPA issue — <specific_cause>"
evidence:
  - type: hpa_status
    content: "<HPA describe output>"
  - type: metrics
    content: "<current vs target metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix metrics pipeline or HPA configuration"
  long_term: "Right-size resource requests, implement proper scaling policies"
```

## Safety Ratings
- GREEN: read-only (`kubectl get hpa`, `kubectl describe hpa`, `kubectl get pods -n kube-system -l k8s-app=metrics-server`, `kubectl top pods`)
- YELLOW: state-changing recoverable (`kubectl apply` HPA configuration, `kubectl apply` metrics-server installation, adjust HPA min/max replicas, update resource requests on pods)
- RED: destructive/irreversible (`kubectl delete hpa` in production causing uncontrolled scaling, removing metrics-server affecting all HPAs cluster-wide)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves modifying HPA scaling limits in production"
- "Remediation requires installing or updating metrics-server in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current HPA configuration before modification: `kubectl get hpa <name> -n <namespace> -o yaml`"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify HPA is reading metrics and scaling correctly after change"
- Revert: "Restore HPA configuration from backup if scaling behavior breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- LOW: "HPA configuration and metrics data"
- LOW: "Pod status and events"

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `kubectl delete namespace` in production without confirmation
- NEVER suggest modifying kube-system resources without backup

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
