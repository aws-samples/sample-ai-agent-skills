---
title: "D1 — Networking Cilium"
description: "Diagnose Cilium CNI networking issues in EKS Anywhere"
status: active
severity: HIGH
triggers: ["pod networking failed", "Cilium error", "CNI not ready", "network policy not working", "pod-to-pod communication failed"]
owner: devops-agent
objective: "Resolve Cilium CNI issues including pod networking, network policies, and connectivity"
context: "Cilium is the default and only supported CNI for EKS Anywhere. It handles pod networking, network policies, and optionally kube-proxy replacement."
---

## Phase 1 — Triage

MUST:
- Check Cilium agent status: `kubectl get pods -n kube-system -l k8s-app=cilium --kubeconfig <workload-kubeconfig>`
- Check Cilium operator: `kubectl get pods -n kube-system -l name=cilium-operator --kubeconfig <workload-kubeconfig>`
- Review Cilium status: `kubectl exec -n kube-system ds/cilium -- cilium status --brief --kubeconfig <workload-kubeconfig>`

SHOULD:
- Check Cilium connectivity: `kubectl exec -n kube-system ds/cilium -- cilium connectivity test --kubeconfig <workload-kubeconfig>`
- Review Cilium agent logs: `kubectl logs -n kube-system ds/cilium --tail=200 --kubeconfig <workload-kubeconfig>`
- Check node-to-node connectivity and tunnel status

MAY:
- Review Cilium endpoint status: `kubectl exec -n kube-system ds/cilium -- cilium endpoint list`
- Check BPF map status for advanced debugging

## Phase 2 — Remediate

MUST:
- Fix Cilium agent crashes (resource limits, kernel compatibility, mount issues)
- Resolve pod CIDR conflicts with existing network infrastructure
- Ensure Cilium configuration in cluster spec matches network requirements

SHOULD:
- Restart Cilium agents on affected nodes: `kubectl rollout restart ds/cilium -n kube-system`
- Verify network policy enforcement mode matches expectations (default, always, never)

MAY:
- Adjust Cilium resource limits if agents are OOMKilled
- Enable Hubble for network flow visibility

## Safety Ratings

```
safety_ratings:
  - "kubectl get pods cilium: GREEN — read-only status check"
  - "cilium status/endpoint list: GREEN — read-only Cilium diagnostics"
  - "kubectl rollout restart ds/cilium: YELLOW — briefly disrupts pod networking during restart"
  - "cilium connectivity test: GREEN — non-destructive connectivity validation"
```

## Escalation Conditions

- Cilium agents crash-loop on all nodes causing cluster-wide networking failure
- Pod CIDR conflicts with corporate network that cannot be resolved without network team
- Kernel version incompatibility with Cilium BPF requirements
- Network policies block critical cluster communication (DNS, API server)

## Data Sensitivity

- `kubectl get pods cilium`: LOW — returns pod status
- `cilium status`: LOW — returns Cilium health and configuration
- `cilium endpoint list`: MEDIUM — reveals pod network identities and IPs
- `Cilium agent logs`: MEDIUM — may contain network flow details and policy decisions

## Prohibited Actions

- NEVER replace Cilium with another CNI — it is the only supported CNI for EKS Anywhere
- NEVER delete the Cilium DaemonSet — this will cause complete networking failure
- NEVER modify Cilium BPF maps directly — use Cilium CLI or configuration
- NEVER disable network policy enforcement in production without security team approval

## Phase 3 — Rollback

1. If Cilium configuration was changed, revert to previous cluster spec and re-apply
2. If Cilium agents were restarted, verify all endpoints are healthy: `cilium endpoint list`
3. If network policies were modified, restore from Git or backup
4. Verify pod-to-pod and pod-to-service connectivity is restored
5. Check that DNS resolution works: `kubectl run test --image=busybox --rm -it -- nslookup kubernetes`

## Common Issues

- symptoms: "Cilium agent CrashLoopBackOff — mount BPF filesystem failed"
  diagnosis: "Node kernel does not support BPF or BPF filesystem is not mounted."
  resolution: "Verify kernel version supports BPF (4.9+). Check /sys/fs/bpf mount."

## Output Format

```yaml
root_cause: "networking_cilium — <specific_cause>"
evidence:
  - type: cilium_error
    content: "<error details from cilium status or logs>"
severity: HIGH
mitigation:
  immediate: "Restore Cilium functionality"
  long_term: "Monitor Cilium health and set up alerts"
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
