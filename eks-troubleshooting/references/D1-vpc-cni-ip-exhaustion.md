---
title: "D1 — VPC CNI / IP Exhaustion"
description: "Diagnose and resolve VPC CNI plugin issues and IP address exhaustion"
status: active
severity: CRITICAL
triggers:
  - "failed to assign an IP address"
  - "ip exhaustion"
  - "ENI"
  - "aws-node"
  - "ipamd"
owner: devops-agent
objective: "Identify VPC CNI issues or IP exhaustion and restore pod networking"
context: "The VPC CNI plugin (aws-node DaemonSet) assigns real VPC IPs to pods via ENI secondary IPs. Each instance type has ENI and IP-per-ENI limits. When subnets run out of IPs or ENI limits are reached, new pods cannot get IPs and stay Pending."
---

## Phase 1 — Triage

MUST:
- Check aws-node DaemonSet status: `kubectl get ds aws-node -n kube-system`
- Check aws-node pod logs: `kubectl logs -n kube-system -l k8s-app=aws-node -c aws-node --tail=50`
- Check IPAMD (IP Address Management Daemon) logs: `kubectl logs -n kube-system -l k8s-app=aws-node -c aws-vpc-cni-init`
- Check subnet available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{Id:SubnetId,AvailableIPs:AvailableIpAddressCount,CIDR:CidrBlock}'`
- Check ENI allocation per node: `kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.pods}{"\n"}{end}'`

SHOULD:
- Check VPC CNI version: `kubectl describe ds aws-node -n kube-system | grep Image`
- Check if prefix delegation is enabled: `kubectl get ds aws-node -n kube-system -o jsonpath='{.spec.template.spec.containers[0].env}' | grep ENABLE_PREFIX_DELEGATION`
- Check WARM_ENI_TARGET and WARM_IP_TARGET settings
- Verify instance type ENI limits: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].NetworkInfo.{MaxENIs:MaximumNetworkInterfaces,IPv4PerENI:Ipv4AddressesPerInterface}'`

MAY:
- Check for leaked ENIs: `aws ec2 describe-network-interfaces --filters Name=description,Values="*amazon-k8s*" --query 'NetworkInterfaces[?Status==\`available\`]'`
- Check CNI metrics if Container Insights is enabled
- Review VPC CNI configuration: `kubectl get configmap amazon-vpc-cni -n kube-system -o yaml`

## Phase 2 — Remediate

MUST:
- If subnet IP exhaustion: add secondary CIDR to VPC and create new subnets, or enable prefix delegation
- If CNI pod is crashing: check CNI logs, restart the aws-node pod: `kubectl delete pod -n kube-system -l k8s-app=aws-node --field-selector spec.nodeName=<node>`
- If ENI limit reached: use larger instance types or enable prefix delegation
- Enable prefix delegation for higher IP density: set `ENABLE_PREFIX_DELEGATION=true` and `WARM_PREFIX_TARGET=1` on the aws-node DaemonSet

SHOULD:
- Use custom networking to assign pod IPs from a different CIDR than node IPs
- Tune WARM_ENI_TARGET and MINIMUM_IP_TARGET for your workload pattern
- Update VPC CNI to the latest compatible version

MAY:
- Consider using secondary VPC CIDR (100.64.0.0/16) for pod networking
- Implement IP address monitoring and alerting on subnet utilization

## Common Issues

- symptoms: "Pods stuck Pending with 'failed to assign an IP address to the container'"
  diagnosis: "Subnet has no available IPs or node has reached ENI/IP limit."
  resolution: "Check subnet available IPs. Enable prefix delegation or add larger subnets."

- symptoms: "aws-node pod in CrashLoopBackOff"
  diagnosis: "VPC CNI plugin crash — often due to IAM permissions, incompatible version, or corrupted state."
  resolution: "Check aws-node logs. Ensure node role has AmazonEKS_CNI_Policy. Update CNI version."

- symptoms: "Nodes show max pods much lower than expected"
  diagnosis: "Max pods is calculated from ENI limits. Small instance types have low limits."
  resolution: "Use larger instance types or enable prefix delegation (increases max pods significantly)."

## Output Format

```yaml
root_cause: "VPC CNI / IP exhaustion — <specific_cause>"
evidence:
  - type: subnet_ips
    content: "<available IPs per subnet>"
  - type: cni_logs
    content: "<relevant aws-node log entries>"
severity: CRITICAL
mitigation:
  immediate: "Add subnet capacity or enable prefix delegation"
  long_term: "Implement subnet IP monitoring, use prefix delegation, plan CIDR allocation"
```

## Safety Ratings
YELLOW — Triage is read-only (`kubectl logs`, `aws ec2 describe-subnets`). Remediation involves modifying the aws-node DaemonSet environment variables (ENABLE_PREFIX_DELEGATION), restarting CNI pods in kube-system, and VPC CIDR changes — these affect cluster-wide pod networking.

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
