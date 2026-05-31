---
title: "B1 — Node NotReady"
description: "Diagnose and resolve EKS worker nodes in NotReady state"
status: active
severity: CRITICAL
triggers:
  - "NotReady"
  - "node not ready"
  - "kubelet stopped"
  - "node condition"
owner: devops-agent
objective: "Identify why the node is NotReady and restore it to Ready state"
context: "A node enters NotReady when the kubelet stops reporting to the API server. Common causes: kubelet failure, VPC CNI crash, disk pressure, memory pressure, certificate expiry, or network connectivity loss to the API server. NotReady nodes cause pod evictions after the pod-eviction-timeout (default 5 minutes)."
---

## Phase 1 — Triage

MUST:
- Check node status and conditions: `kubectl get nodes -o wide`
- Describe the NotReady node: `kubectl describe node <node-name>`
- Look for conditions: Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable
- Check recent events on the node: `kubectl get events --field-selector involvedObject.name=<node-name>`
- Check node group health: `aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <ng> --query 'nodegroup.health'`

SHOULD:
- If SSM is available, check kubelet status: `systemctl status kubelet`
- Check kubelet logs: `journalctl -u kubelet --no-pager -n 100`
- Check VPC CNI plugin status: `kubectl get pods -n kube-system -l k8s-app=aws-node -o wide`
- Check the node's EC2 instance status: `aws ec2 describe-instance-status --instance-ids <instance-id>`

MAY:
- Check disk usage on the node: `df -h` (via SSM)
- Check memory usage: `free -m` (via SSM)
- Check if the node can reach the API server: `curl -k https://<api-server-endpoint>/healthz` (via SSM)
- Check CNI plugin logs: `kubectl logs -n kube-system <aws-node-pod> -c aws-node`

## Phase 2 — Remediate

MUST:
- If kubelet is crashed: restart kubelet via SSM: `systemctl restart kubelet`
- If disk pressure: clean up unused images and containers: `crictl rmi --prune`
- If VPC CNI failure: restart the aws-node pod: `kubectl delete pod <aws-node-pod> -n kube-system`
- If the node is unrecoverable: cordon, drain, and terminate: `kubectl cordon <node>; kubectl drain <node> --ignore-daemonsets --delete-emptydir-data; aws ec2 terminate-instances --instance-ids <id>`

SHOULD:
- Check if the issue affects multiple nodes (cluster-wide problem vs single node)
- For managed node groups: the ASG will replace terminated instances automatically
- Monitor node recovery: `kubectl get nodes -w`

MAY:
- Increase node group size to maintain capacity while troubleshooting
- Check CloudWatch metrics for the EC2 instance (CPU, network, disk)

## Common Issues

- symptoms: "Node shows NotReady, kubelet logs show 'failed to get cgroup stats'"
  diagnosis: "Kubelet cannot read cgroup information, often due to disk pressure or corrupted cgroup state."
  resolution: "Restart kubelet. If persistent, drain and replace the node."

- symptoms: "Node NotReady with NetworkUnavailable condition True"
  diagnosis: "VPC CNI plugin (aws-node) has failed on this node."
  resolution: "Delete the aws-node pod on the affected node to trigger restart. Check CNI logs for errors."

- symptoms: "All nodes go NotReady simultaneously"
  diagnosis: "Cluster-wide issue — likely API server connectivity, security group change, or NACL modification."
  resolution: "Check cluster endpoint access, security groups, and NACLs. Verify control plane health."

- symptoms: "Node NotReady after cluster upgrade"
  diagnosis: "Kubelet version skew — node is more than 2 minor versions behind the control plane."
  resolution: "Update the node group to match the control plane version."

## Output Format

```yaml
root_cause: "Node NotReady — <specific_cause>"
evidence:
  - type: node_conditions
    content: "<kubectl describe node output>"
  - type: kubelet_logs
    content: "<relevant kubelet log entries>"
severity: CRITICAL
mitigation:
  immediate: "Restart kubelet/CNI or replace the node"
  long_term: "Implement node health monitoring and auto-remediation via node problem detector"
```

## Safety Ratings
RED — Remediation involves node drain (`kubectl drain`), node cordoning, instance termination, and restarting kube-system components (kubelet, aws-node). Draining nodes affects running production workloads and can cause service disruption.

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
