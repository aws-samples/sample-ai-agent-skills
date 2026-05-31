---
title: "A1 — Cluster Creation Failures"
description: "Diagnose and resolve EKS cluster creation failures"
status: active
severity: HIGH
triggers:
  - "cluster creation failed"
  - "CREATE_FAILED"
  - "ResourceInUseException"
  - "InvalidParameterException"
owner: devops-agent
objective: "Identify why the EKS cluster failed to create and resolve the issue"
context: "EKS cluster creation involves provisioning a managed control plane, setting up ENIs in the specified subnets, and configuring the cluster security group. Failures typically stem from IAM role issues, VPC/subnet misconfiguration, or service limits."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws eks describe-cluster --name <cluster> --query 'cluster.{status:status,platformVersion:platformVersion}'`
- Check CloudTrail for CreateCluster event errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCluster --max-results 5`
- Verify the cluster IAM role exists and has the correct trust policy: `aws iam get-role --role-name <cluster-role>`
- Verify the role has `AmazonEKSClusterPolicy` attached: `aws iam list-attached-role-policies --role-name <cluster-role>`

SHOULD:
- Check subnet configuration — at least 2 subnets in different AZs: `aws ec2 describe-subnets --subnet-ids <subnet-1> <subnet-2>`
- Verify subnets have available IP addresses (EKS needs ENIs in the subnets)
- Check the security group allows required traffic: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check EKS service limits: `aws service-quotas get-service-quota --service-code eks --quota-code L-1194D53C`

MAY:
- If using eksctl, check CloudFormation stack events: `aws cloudformation describe-stack-events --stack-name eksctl-<cluster>-cluster`
- Check if the VPC has DNS support enabled: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`

## Phase 2 — Remediate

MUST:
- If IAM role issue: create the role with `eks.amazonaws.com` trust policy and attach `AmazonEKSClusterPolicy`
- If subnet issue: ensure at least 2 subnets in different AZs with available IPs
- If security group issue: ensure the SG allows outbound HTTPS (443) to AWS services
- If quota exceeded: request a limit increase via Service Quotas

SHOULD:
- Use subnets with at least /24 CIDR (16 IPs minimum for EKS ENIs)
- Enable VPC DNS support and DNS hostnames
- Tag subnets appropriately for load balancer discovery

MAY:
- Delete the failed cluster and recreate: `aws eks delete-cluster --name <cluster>`
- Use eksctl for simplified cluster creation with validated defaults

## Common Issues

- symptoms: "InvalidParameterException: The provided role doesn't have the Amazon EKS Managed Policies"
  diagnosis: "Cluster IAM role missing required managed policy."
  resolution: "Attach AmazonEKSClusterPolicy to the cluster role."

- symptoms: "ResourceInUseException: Cluster already exists"
  diagnosis: "A cluster with the same name exists (possibly in FAILED state)."
  resolution: "Delete the existing cluster first, then recreate."

- symptoms: "Cluster stuck in CREATING for more than 20 minutes"
  diagnosis: "Usually a VPC/subnet issue — EKS cannot create ENIs in the specified subnets."
  resolution: "Check subnet available IPs and security group rules. Ensure subnets are in different AZs."

- symptoms: "UnsupportedAvailabilityZoneException"
  diagnosis: "One or more specified AZs don't support EKS."
  resolution: "Use different subnets in supported AZs. Check EKS availability per region."

## Output Format

```yaml
root_cause: "Cluster creation failure — <specific_cause>"
evidence:
  - type: cloudtrail_event
    content: "<CreateCluster error details>"
  - type: iam_role
    content: "<role configuration or missing policy>"
severity: HIGH
mitigation:
  immediate: "Fix the identified configuration issue and retry cluster creation"
  long_term: "Use IaC (CloudFormation, Terraform, eksctl) with validated configurations"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`aws eks describe-cluster`, `aws cloudtrail lookup-events`, `aws iam get-role`). Remediation involves creating/fixing IAM roles and VPC configuration outside the cluster. Cluster deletion of a failed cluster is low-risk.

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
