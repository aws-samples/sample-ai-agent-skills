---
title: "A2 — Cluster Endpoint Access Issues"
description: "Diagnose and resolve EKS API server endpoint access problems"
status: active
severity: CRITICAL
triggers:
  - "Unable to connect to the server"
  - "couldn't get current server API group list"
  - "endpoint unreachable"
  - "connection refused"
owner: devops-agent
objective: "Restore access to the EKS cluster API server endpoint"
context: "EKS clusters have configurable endpoint access: public, private, or both. Misconfigured endpoint access is a common cause of kubectl failures. The API server endpoint is an NLB fronting the managed control plane."
---

## Phase 1 — Triage

MUST:
- Check cluster endpoint configuration: `aws eks describe-cluster --name <cluster> --query 'cluster.resourcesVpcConfig.{endpointPublicAccess:endpointPublicAccess,endpointPrivateAccess:endpointPrivateAccess,publicAccessCidrs:publicAccessCidrs}'`
- Verify kubeconfig is correct: `kubectl config current-context`
- Refresh kubeconfig: `aws eks update-kubeconfig --name <cluster> --region <region>`
- Test API server connectivity: `kubectl cluster-info`
- Check if your IP is in the public access CIDR list (if public access is restricted)

SHOULD:
- Verify AWS credentials: `aws sts get-caller-identity`
- Check if the IAM identity has eks:DescribeCluster permission
- If private endpoint only: verify you're connecting from within the VPC or via VPN/Direct Connect
- Check DNS resolution of the cluster endpoint: `nslookup <cluster-endpoint>`

MAY:
- Check VPC security groups for the cluster: `aws eks describe-cluster --name <cluster> --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId'`
- Verify the cluster security group allows inbound 443 from your network
- Check for NACLs blocking traffic to the API server ENIs

## Phase 2 — Remediate

MUST:
- If public access is disabled and you're outside the VPC: enable public access or connect via VPN
- If public access CIDRs are restricted: add your IP to the allowed list: `aws eks update-cluster-config --name <cluster> --resources-vpc-config publicAccessCidrs=<your-cidr>`
- If kubeconfig is stale: regenerate with `aws eks update-kubeconfig --name <cluster> --region <region>`

SHOULD:
- For production: use public+private access with restricted public CIDRs
- Ensure the cluster security group allows 443 from worker node security groups
- If using private endpoint: configure VPC DNS resolution for the cluster endpoint

MAY:
- Set up a bastion host or Cloud9 environment in the VPC for private-only clusters
- Use AWS Systems Manager Session Manager to tunnel kubectl through a VPC instance

## Common Issues

- symptoms: "Unable to connect to the server: dial tcp: i/o timeout"
  diagnosis: "Network path to API server is blocked. Either endpoint access config or network routing."
  resolution: "Check endpoint public/private settings. If private-only, ensure VPC connectivity."

- symptoms: "error: You must be logged in to the server (Unauthorized)"
  diagnosis: "IAM identity is not mapped in aws-auth ConfigMap or credentials are expired."
  resolution: "Check aws-auth ConfigMap. Verify AWS credentials with `aws sts get-caller-identity`."

- symptoms: "Unable to connect to the server: x509: certificate signed by unknown authority"
  diagnosis: "kubeconfig has wrong cluster CA data, or a proxy is intercepting TLS."
  resolution: "Regenerate kubeconfig: `aws eks update-kubeconfig --name <cluster>`."

## Output Format

```yaml
root_cause: "Endpoint access — <specific_cause>"
evidence:
  - type: cluster_config
    content: "<endpoint access settings>"
  - type: connectivity_test
    content: "<kubectl cluster-info output or error>"
severity: CRITICAL
mitigation:
  immediate: "Restore API server access via endpoint config or network path fix"
  long_term: "Use public+private endpoint with restricted CIDRs for production clusters"
```

## Safety Ratings
RED — Remediation involves modifying cluster endpoint access configuration (`aws eks update-cluster-config`), which can lock out all users if misconfigured. Changes to public access CIDRs and endpoint settings are high-risk in production.

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
