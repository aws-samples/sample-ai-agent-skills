---
title: "C3 — OOMKilled"
description: "Diagnose and resolve containers killed due to out-of-memory conditions"
status: active
severity: HIGH
triggers:
  - "OOMKilled"
  - "exit code 137"
  - "memory limit"
  - "out of memory"
owner: devops-agent
objective: "Identify the memory issue and prevent OOMKilled events"
context: "OOMKilled occurs when a container exceeds its memory limit (cgroup OOM) or the node runs out of memory (system OOM). Exit code 137 = SIGKILL from OOM killer. The container's memory limit is a hard cap enforced by the kernel cgroup."
---

## Phase 1 — Triage

MUST:
- Confirm OOMKilled: `kubectl describe pod <pod> -n <namespace>` — look for "OOMKilled" in Last State
- Check container memory limits: `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].resources.limits.memory}'`
- Check actual memory usage before kill (if metrics-server installed): `kubectl top pod <pod> -n <namespace>`
- Check node memory pressure: `kubectl describe node <node> | grep -A5 MemoryPressure`

SHOULD:
- Check if the pod has memory requests set (for scheduling): `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].resources.requests.memory}'`
- Review CloudWatch Container Insights for memory usage trends
- Check if the OOMKill is consistent (every restart) or intermittent (load-dependent)
- Look at previous container logs for memory-related errors: `kubectl logs <pod> -n <namespace> --previous`

MAY:
- Profile the application's memory usage in a test environment
- Check for known memory leaks in the application or its dependencies
- Check node-level memory: `kubectl top nodes`

## Phase 2 — Remediate

MUST:
- If memory limit too low: increase the memory limit in the pod spec
- If memory leak: fix the application code (this is the real fix, not just increasing limits)
- If no memory limit set: add appropriate limits to prevent unbounded growth
- If node-level OOM: add more nodes or use larger instance types

SHOULD:
- Set memory requests equal to limits for guaranteed QoS class (prevents eviction)
- Use Vertical Pod Autoscaler (VPA) to right-size memory requests/limits
- Implement application-level memory monitoring and alerting

MAY:
- Use Java: set `-XX:MaxRAMPercentage=75` to respect container limits
- Use Node.js: set `--max-old-space-size` to match container limits
- Enable Container Insights for historical memory usage data

## Common Issues

- symptoms: "Pod OOMKilled with 256Mi limit, application needs 512Mi at peak"
  diagnosis: "Memory limit is too low for the application's peak usage."
  resolution: "Increase memory limit to accommodate peak usage with headroom."

- symptoms: "Pod OOMKilled after running for hours, memory usage grows linearly"
  diagnosis: "Memory leak in the application."
  resolution: "Fix the memory leak. As a temporary workaround, increase limits and set up periodic restarts."

- symptoms: "Java application OOMKilled despite JVM heap being within limits"
  diagnosis: "JVM uses memory beyond heap (metaspace, thread stacks, native memory). Container limit must account for total JVM memory."
  resolution: "Set container limit = heap + metaspace + thread stacks + ~25% overhead. Use `-XX:MaxRAMPercentage=75`."

## Output Format

```yaml
root_cause: "OOMKilled — <specific_cause>"
evidence:
  - type: pod_status
    content: "<OOMKilled state from describe pod>"
  - type: memory_usage
    content: "<memory limit vs actual usage>"
severity: HIGH
mitigation:
  immediate: "Increase memory limits or fix memory leak"
  long_term: "Right-size with VPA, implement memory monitoring, fix application memory issues"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`kubectl describe pod`, `kubectl top pod`, `kubectl get`). Remediation involves adjusting memory limits in pod specs, which is a non-destructive change scoped to the affected workload.

## Escalation Conditions
- Remediation requires editing aws-auth ConfigMap in production
- Fix involves modifying RBAC ClusterRoleBindings
- Node group changes affect running production workloads
- Cluster endpoint access changes could lock out users
- IRSA trust policy modifications are needed

## Rollback
1. Before aws-auth changes: `kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-backup.yaml`
2. Before RBAC changes: Save current bindings with `kubectl get clusterrolebinding <name> -o yaml`
3. After change: Verify cluster access and workload health
4. If change causes lockout: Use cluster creator credentials or restore from backup
5. Cleanup: Remove any temporary RBAC bindings or test service accounts

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `kubectl get configmap aws-auth` | HIGH | IAM-to-RBAC mappings — redact |
| `kubectl get sa -A -o yaml` | HIGH | Service account annotations with role ARNs — redact |
| `kubectl get events` | MEDIUM | Cluster events — summarize |
| `kubectl get nodes/pods` | LOW | Resource status — safe to include |

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `--force` delete of namespaces or persistent volumes
- NEVER suggest modifying kube-system resources without backup
- ALWAYS backup aws-auth before any modification
- ALWAYS use specific RBAC roles instead of cluster-admin

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
