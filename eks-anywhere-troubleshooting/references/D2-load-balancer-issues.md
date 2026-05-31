---
title: "D2 — Load Balancer Issues"
description: "Diagnose load balancer and service exposure issues in EKS Anywhere"
status: active
severity: HIGH
triggers: ["LoadBalancer pending", "external IP not assigned", "kube-vip error", "MetalLB issue", "service not reachable"]
owner: devops-agent
objective: "Resolve load balancer issues including kube-vip, MetalLB, and service exposure"
context: "EKS Anywhere uses kube-vip for control plane VIP and optionally MetalLB or other bare-metal load balancers for service type LoadBalancer. Cloud load balancers (ALB, NLB) are not available on-premises."
---

## Phase 1 — Triage

MUST:
- Check services with pending LoadBalancer: `kubectl get svc -A --field-selector spec.type=LoadBalancer --kubeconfig <workload-kubeconfig>`
- Check kube-vip status: `kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-static --kubeconfig <workload-kubeconfig>`
- Verify control plane VIP is reachable: `ping <control-plane-endpoint>`

SHOULD:
- Check MetalLB or load balancer controller pods: `kubectl get pods -n metallb-system --kubeconfig <workload-kubeconfig>`
- Review IP address pool configuration: `kubectl get ipaddresspools -A --kubeconfig <workload-kubeconfig>`
- Check ARP/BGP announcements for assigned IPs

MAY:
- Verify network allows ARP or BGP from cluster nodes
- Check for IP address conflicts on the network

## Phase 2 — Remediate

MUST:
- Fix IP address pool exhaustion by expanding the pool or releasing unused IPs
- Resolve kube-vip failures affecting control plane accessibility
- Ensure load balancer controller is running and healthy

SHOULD:
- Verify assigned IPs are in a routable subnet accessible to clients
- Check that ARP responses or BGP advertisements are reaching the network

MAY:
- Configure additional IP address pools for different services
- Set up L2 or BGP mode based on network infrastructure

## Safety Ratings

```
safety_ratings:
  - "kubectl get svc/pods: GREEN — read-only status check"
  - "ping control-plane-endpoint: GREEN — non-destructive connectivity test"
  - "kubectl get ipaddresspools: GREEN — read-only configuration check"
  - "IP pool modification: YELLOW — affects service IP assignment"
```

## Escalation Conditions

- Control plane VIP is unreachable making the cluster inaccessible
- IP address pool completely exhausted with no available IPs
- Network team intervention required for ARP/BGP configuration
- kube-vip pod crashes repeatedly on control plane nodes

## Data Sensitivity

- `kubectl get svc`: LOW — returns service metadata and IPs
- `kubectl get ipaddresspools`: LOW — returns IP range configuration
- `kube-vip logs`: MEDIUM — may contain network interface and ARP details
- `MetalLB logs`: MEDIUM — may contain BGP peer and network topology information

## Prohibited Actions

- NEVER delete kube-vip pods on control plane nodes — this makes the cluster unreachable
- NEVER assign the control plane VIP to a LoadBalancer service
- NEVER modify IP address pools without verifying no conflicts with existing network assignments
- NEVER switch load balancer mode (L2 to BGP) on a production cluster without planning

## Phase 3 — Rollback

1. If IP pool was modified incorrectly, revert to previous configuration
2. If kube-vip was disrupted, verify control plane VIP is reassigned: `ping <control-plane-endpoint>`
3. If MetalLB configuration was changed, restore from Git or backup
4. Verify all LoadBalancer services have assigned IPs: `kubectl get svc -A`
5. Confirm external access to services is restored

## Common Issues

- symptoms: "Service stuck in Pending — no external IP assigned"
  diagnosis: "No load balancer controller installed or IP address pool exhausted."
  resolution: "Install MetalLB or expand IP address pool. Verify controller pods are running."

## Output Format

```yaml
root_cause: "load_balancer — <specific_cause>"
evidence:
  - type: lb_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore load balancer functionality"
  long_term: "Monitor IP pool utilization and set up alerts"
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
