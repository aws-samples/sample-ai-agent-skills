---
title: "B2 — Kubelet Issues"
description: "Diagnose Bottlerocket kubelet and EKS node registration failures"
status: active
severity: HIGH
triggers: ["node not ready", "kubelet not running", "node registration failed", "NotReady status", "eks node issues"]
owner: devops-agent
objective: "Restore kubelet connectivity and EKS node registration on Bottlerocket instances"
context: "Bottlerocket EKS variant runs kubelet as a system service. Kubernetes settings are configured through the settings API with TOML user data."
---

## Phase 1 — Triage

MUST:
- Check node status from EKS: `kubectl get nodes -o wide | grep <instance-id>`
- Verify Kubernetes settings: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get settings.kubernetes"`
- Check node group health: `aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <nodegroup> --query 'nodegroup.{Status:status,Health:health}'`

SHOULD:
- Check console output for kubelet errors: `aws ec2 get-console-output --instance-id <instance-id> --latest`
- Verify the aws-auth ConfigMap includes the node IAM role: `kubectl describe configmap aws-auth -n kube-system`

MAY:
- Collect detailed logs: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="logdog"`

## Phase 2 — Remediate

MUST:
- Verify cluster-name, api-server, and cluster-certificate are set correctly in user data
- Ensure the node IAM role is mapped in aws-auth ConfigMap
- Confirm network connectivity to the EKS API server endpoint

SHOULD:
- For private clusters, verify VPC endpoints for EKS, ECR, STS, and S3
- Check that the Kubernetes version in the AMI matches the cluster version

MAY:
- Reboot the instance to restart kubelet: `apiclient reboot`

## Safety Ratings

```
safety_ratings:
  - "kubectl get nodes: GREEN — read-only node status check"
  - "apiclient get settings.kubernetes: GREEN — read-only settings query"
  - "eks describe-nodegroup: GREEN — read-only node group status"
  - "apiclient reboot: RED — causes instance reboot and pod eviction"
```

## Escalation Conditions

- Node remains NotReady after verifying all settings and network connectivity
- Kubelet crashes repeatedly with OOM or certificate errors
- aws-auth ConfigMap changes cause all nodes to lose connectivity
- Cluster version upgrade causes Bottlerocket nodes to become incompatible

## Data Sensitivity

- `kubectl get nodes`: LOW — returns node names and status
- `apiclient get settings.kubernetes`: HIGH — contains cluster certificate and API endpoint
- `aws-auth ConfigMap`: HIGH — contains IAM role mappings for cluster access
- `aws ec2 get-console-output`: MEDIUM — may contain bootstrap secrets

## Prohibited Actions

- NEVER edit kubelet configuration files directly — use the settings API
- NEVER delete the aws-auth ConfigMap — this locks out all nodes
- NEVER cordon and drain a node without verifying pod disruption budgets
- NEVER suggest using Docker commands — Bottlerocket uses containerd with kubelet

## Phase 3 — Rollback

1. If settings were changed, revert: `apiclient set kubernetes.cluster-name=<original>`
2. If aws-auth was modified incorrectly, restore from backup: `kubectl apply -f aws-auth-backup.yaml`
3. If node group was updated, roll back: `aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <nodegroup> --release-version <previous>`
4. If instance is unrecoverable, terminate and let auto scaling replace it
5. Verify pods are rescheduled to healthy nodes after rollback

## Common Issues

- symptoms: "Node shows NotReady with kubelet stopped posting status"
  diagnosis: "Kubernetes settings (cluster-name, api-server, cluster-certificate) are missing or incorrect in user data."
  resolution: "Verify TOML user data contains correct [settings.kubernetes] values. Terminate and relaunch if user data cannot be changed."

- symptoms: "Node registers but immediately becomes NotReady"
  diagnosis: "Node IAM role is not mapped in aws-auth ConfigMap or VPC CNI plugin fails."
  resolution: "Add node IAM role to aws-auth ConfigMap and verify VPC CNI DaemonSet is running."

## Output Format

```yaml
root_cause: "kubelet — <specific_cause>"
evidence:
  - type: node_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix kubelet settings or node registration"
  long_term: "Validate user data and aws-auth in IaC"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
