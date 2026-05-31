---
title: "C2 — CrashLoopBackOff"
description: "Diagnose and resolve pods in CrashLoopBackOff state"
status: active
severity: HIGH
triggers:
  - "CrashLoopBackOff"
  - "BackOff"
  - "restarting"
  - "exit code"
owner: devops-agent
objective: "Identify why the container is crashing and fix the root cause"
context: "CrashLoopBackOff means the container starts, crashes, and Kubernetes keeps restarting it with exponential backoff (10s, 20s, 40s, up to 5 minutes). The root cause is always in the container itself — application error, missing config, failed health check, or resource constraints."
---

## Phase 1 — Triage

MUST:
- Check pod status and restart count: `kubectl get pod <pod> -n <namespace>`
- Get container logs (current attempt): `kubectl logs <pod> -n <namespace> -c <container>`
- Get previous crash logs: `kubectl logs <pod> -n <namespace> -c <container> --previous`
- Describe the pod for events and exit codes: `kubectl describe pod <pod> -n <namespace>`
- Note the exit code: 0=success, 1=app error, 137=OOMKilled/SIGKILL, 139=segfault, 143=SIGTERM

SHOULD:
- Check if the container has liveness/readiness probes that might be failing
- Check environment variables and ConfigMaps/Secrets: `kubectl get pod <pod> -n <namespace> -o yaml`
- Verify the container image exists and is pullable
- Check resource limits — container might be hitting memory limits

MAY:
- Run the container interactively to debug: `kubectl run debug --image=<image> -it --rm -- /bin/sh`
- Check init containers if they're failing: `kubectl logs <pod> -n <namespace> -c <init-container>`
- Check if the issue started after a deployment: `kubectl rollout history deployment/<name> -n <namespace>`

## Phase 2 — Remediate

MUST:
- If application error (exit code 1): fix the application code or configuration
- If OOMKilled (exit code 137): increase memory limits or fix memory leak
- If liveness probe failing: fix the probe configuration or the health endpoint
- If missing config: fix ConfigMap, Secret, or environment variable references

SHOULD:
- Roll back to the last working version: `kubectl rollout undo deployment/<name> -n <namespace>`
- Add or fix readiness probes to prevent traffic to unhealthy pods
- Review resource requests and limits

MAY:
- Use ephemeral debug containers: `kubectl debug <pod> -n <namespace> -it --image=busybox`
- Check if the issue is environment-specific (works in staging, fails in prod)

## Common Issues

- symptoms: "Container exits with code 1, logs show 'connection refused' to database"
  diagnosis: "Application cannot connect to a dependency (database, cache, API)."
  resolution: "Check service connectivity, DNS resolution, and network policies."

- symptoms: "Container exits with code 137, no application logs"
  diagnosis: "OOMKilled — container exceeded its memory limit."
  resolution: "Increase memory limits or investigate memory leak. Check: `kubectl describe pod` for OOMKilled reason."

- symptoms: "Container starts but liveness probe fails after 30 seconds"
  diagnosis: "Application takes longer to start than the liveness probe initialDelaySeconds."
  resolution: "Increase initialDelaySeconds or use a startup probe for slow-starting containers."

- symptoms: "CrashLoopBackOff after ConfigMap or Secret update"
  diagnosis: "Updated config has invalid values or missing required keys."
  resolution: "Check the ConfigMap/Secret contents. Roll back the config change if needed."

## Output Format

```yaml
root_cause: "CrashLoopBackOff — <specific_cause>"
evidence:
  - type: container_logs
    content: "<relevant log entries>"
  - type: exit_code
    content: "<exit code and meaning>"
severity: HIGH
mitigation:
  immediate: "Fix the crash cause or roll back to working version"
  long_term: "Add proper health checks, resource limits, and deployment rollback strategies"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`kubectl logs`, `kubectl describe pod`, `kubectl get`). Remediation involves application-level fixes (config, resource limits) or rollback (`kubectl rollout undo`) which is a safe, built-in Kubernetes operation.

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
