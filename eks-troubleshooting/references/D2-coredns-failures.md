---
title: "D2 — CoreDNS Failures"
description: "Diagnose and resolve CoreDNS failures causing DNS resolution issues in the cluster"
status: active
severity: CRITICAL
triggers:
  - "CoreDNS"
  - "DNS"
  - "name resolution"
  - "could not resolve"
  - "kube-dns"
owner: devops-agent
objective: "Restore DNS resolution in the cluster by fixing CoreDNS"
context: "CoreDNS runs as a Deployment (default 2 replicas) in kube-system. It provides DNS resolution for all service discovery in the cluster. If CoreDNS fails, pods cannot resolve service names, and most cluster networking breaks. CoreDNS health depends on node health."
---

## Phase 1 — Triage

MUST:
- Check CoreDNS pod status: `kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide`
- Check CoreDNS pod logs: `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50`
- Describe CoreDNS pods for events: `kubectl describe pods -n kube-system -l k8s-app=kube-dns`
- Check the kube-dns service: `kubectl get svc kube-dns -n kube-system`
- Test DNS resolution from a pod: `kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never -- nslookup kubernetes.default`

SHOULD:
- Check CoreDNS ConfigMap: `kubectl get configmap coredns -n kube-system -o yaml`
- Check CoreDNS deployment replicas: `kubectl get deployment coredns -n kube-system`
- Verify the nodes hosting CoreDNS pods are healthy
- Check CoreDNS resource usage: `kubectl top pods -n kube-system -l k8s-app=kube-dns`

MAY:
- Check CoreDNS add-on version: `aws eks describe-addon --cluster-name <cluster> --addon-name coredns`
- Enable CoreDNS logging for debugging: add `log` plugin to the Corefile
- Check for DNS policy on affected pods: `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.dnsPolicy}'`

## Phase 2 — Remediate

MUST:
- If CoreDNS pods are not running: check why (node issues, resource constraints, scheduling)
- If CoreDNS is OOMKilled: increase memory limits in the CoreDNS deployment
- If CoreDNS ConfigMap is corrupted: restore the default Corefile configuration
- If nodes hosting CoreDNS are unhealthy: fix the nodes first (see B1)

SHOULD:
- Scale CoreDNS replicas for high-traffic clusters: `kubectl scale deployment coredns -n kube-system --replicas=3`
- Use pod anti-affinity to spread CoreDNS across nodes
- Update CoreDNS via EKS add-on: `aws eks update-addon --cluster-name <cluster> --addon-name coredns --addon-version <version>`

MAY:
- Use NodeLocal DNSCache for improved DNS performance and resilience
- Configure CoreDNS autoscaling based on cluster size
- Add custom DNS entries via the CoreDNS ConfigMap

## Common Issues

- symptoms: "DNS resolution fails for all pods, CoreDNS pods are Pending"
  diagnosis: "CoreDNS pods cannot be scheduled — likely node resource exhaustion or taints."
  resolution: "Check node resources and taints. CoreDNS tolerates only specific taints by default."

- symptoms: "CoreDNS pods running but DNS queries timeout"
  diagnosis: "CoreDNS is overloaded or the Corefile has a misconfiguration."
  resolution: "Check CoreDNS logs for errors. Scale up replicas. Verify Corefile syntax."

- symptoms: "DNS works for Kubernetes services but not external domains"
  diagnosis: "CoreDNS forward plugin misconfigured or VPC DNS resolver unreachable."
  resolution: "Check Corefile forward directive. Ensure VPC DNS (169.254.169.253) is reachable."

- symptoms: "Intermittent DNS failures under load"
  diagnosis: "CoreDNS replicas insufficient for the query volume, or conntrack table full."
  resolution: "Scale CoreDNS replicas. Consider NodeLocal DNSCache. Check conntrack limits."

## Output Format

```yaml
root_cause: "CoreDNS failure — <specific_cause>"
evidence:
  - type: coredns_pods
    content: "<pod status and events>"
  - type: dns_test
    content: "<nslookup result>"
severity: CRITICAL
mitigation:
  immediate: "Restore CoreDNS pods to running state"
  long_term: "Scale CoreDNS appropriately, use NodeLocal DNSCache, monitor DNS latency"
```

## Safety Ratings
YELLOW — Triage is read-only (`kubectl get/describe/logs`). Remediation involves scaling CoreDNS (`kubectl scale`), updating the CoreDNS ConfigMap, and updating the CoreDNS add-on — changes to this critical kube-system component affect cluster-wide DNS resolution.

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
