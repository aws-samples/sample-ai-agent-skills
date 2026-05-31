---
title: "D3 — Service Connectivity"
description: "Diagnose and resolve Kubernetes service connectivity issues (ClusterIP, NodePort, LoadBalancer)"
status: active
severity: HIGH
triggers:
  - "service unreachable"
  - "connection refused"
  - "no endpoints"
  - "service not working"
  - "LoadBalancer pending"
owner: devops-agent
objective: "Identify why the Kubernetes service is not routing traffic and fix the connectivity issue"
context: "Kubernetes services route traffic to pods via label selectors. ClusterIP provides internal access, NodePort exposes on node ports, LoadBalancer creates an AWS ELB. Common issues: selector mismatch, no ready endpoints, security group rules, or load balancer configuration."
---

## Phase 1 — Triage

MUST:
- Check service details: `kubectl get svc <service> -n <namespace> -o wide`
- Check service endpoints: `kubectl get endpoints <service> -n <namespace>`
- Verify pods match the service selector: `kubectl get pods -n <namespace> -l <selector-key>=<selector-value>`
- Check if target pods are Ready: `kubectl get pods -n <namespace> -l <selector-key>=<selector-value> -o wide`
- For LoadBalancer type: check if external IP/hostname is assigned

SHOULD:
- Test connectivity from within the cluster: `kubectl run test --image=busybox:1.36 --rm -it --restart=Never -- wget -qO- http://<service>.<namespace>.svc.cluster.local:<port>`
- Check kube-proxy status: `kubectl get pods -n kube-system -l k8s-app=kube-proxy`
- For LoadBalancer: check the AWS ELB: `aws elbv2 describe-load-balancers` or `aws elb describe-load-balancers`
- Check security groups for the service's target port

MAY:
- Check network policies that might block traffic: `kubectl get networkpolicy -n <namespace>`
- Check iptables rules on a node (via SSM): `iptables -t nat -L KUBE-SERVICES`
- Verify DNS resolution for the service: `nslookup <service>.<namespace>.svc.cluster.local`

## Phase 2 — Remediate

MUST:
- If no endpoints: fix the service selector to match pod labels, or ensure pods are Ready
- If LoadBalancer stuck in Pending: check AWS Load Balancer Controller logs or cloud-controller-manager
- If connection refused: verify the target port matches the container's listening port
- If security group blocking: add inbound rules for the service port

SHOULD:
- For LoadBalancer: ensure subnet tags are correct (see D4 for ALB-specific issues)
- Verify readiness probes — pods must pass readiness to be added to endpoints
- Check for ExternalTrafficPolicy settings (Local vs Cluster) affecting traffic routing

MAY:
- Use `kubectl port-forward` to test direct pod connectivity
- Check for service mesh (Istio, App Mesh) interference

## Common Issues

- symptoms: "Service has no endpoints"
  diagnosis: "Service selector doesn't match any pod labels, or all matching pods are not Ready."
  resolution: "Compare service selector with pod labels. Check pod readiness."

- symptoms: "LoadBalancer service stuck in Pending with no external IP"
  diagnosis: "Cloud controller or AWS Load Balancer Controller cannot create the ELB."
  resolution: "Check controller logs. Verify IAM permissions and subnet tags."

- symptoms: "Service works from within the cluster but not externally"
  diagnosis: "Security group, NACL, or routing issue for external access."
  resolution: "Check node security groups allow the NodePort range (30000-32767) or ELB security groups."

## Output Format

```yaml
root_cause: "Service connectivity — <specific_cause>"
evidence:
  - type: service_config
    content: "<service details and endpoints>"
  - type: connectivity_test
    content: "<test result from within cluster>"
severity: HIGH
mitigation:
  immediate: "Fix selector, endpoints, or network configuration"
  long_term: "Implement service monitoring and readiness probes"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`kubectl get svc/endpoints`, `kubectl describe`). Remediation involves fixing service selectors, pod labels, and security group rules — changes scoped to the affected service with no cluster-wide impact.

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
