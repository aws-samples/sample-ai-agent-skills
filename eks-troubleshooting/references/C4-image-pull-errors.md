---
title: "C4 — Image Pull Errors"
description: "Diagnose and resolve container image pull failures (ImagePullBackOff, ErrImagePull)"
status: active
severity: HIGH
triggers:
  - "ImagePullBackOff"
  - "ErrImagePull"
  - "image pull"
  - "repository does not exist"
  - "unauthorized"
owner: devops-agent
objective: "Identify why the container image cannot be pulled and fix the issue"
context: "Image pull errors occur when kubelet cannot download the container image. Common causes: wrong image name/tag, private registry without credentials, ECR authentication issues, network connectivity to the registry, or image doesn't exist."
---

## Phase 1 — Triage

MUST:
- Check pod events for the specific error: `kubectl describe pod <pod> -n <namespace>`
- Verify the image name and tag: `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.containers[*].image}'`
- Check if the image exists in the registry (for ECR): `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>`
- Check if imagePullSecrets are configured: `kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.imagePullSecrets}'`

SHOULD:
- For ECR: verify the node IAM role has `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:GetAuthorizationToken`
- Check if ECR repository policy allows access from the cluster's account/role
- Verify network connectivity to the registry (NAT gateway for private subnets, or VPC endpoints)
- Check if the ECR repository is in the same region as the cluster

MAY:
- Test image pull manually on a node: `crictl pull <image>`
- Check for ECR VPC endpoints if nodes are in private subnets without NAT
- Verify cross-account ECR access if the image is in a different account

## Phase 2 — Remediate

MUST:
- If image doesn't exist: fix the image name, tag, or push the image to the registry
- If ECR auth issue: ensure node IAM role has ECR permissions
- If private registry: create and attach imagePullSecret: `kubectl create secret docker-registry <name> --docker-server=<registry> --docker-username=<user> --docker-password=<pass>`
- If network issue: ensure NAT gateway or VPC endpoints for ECR (com.amazonaws.<region>.ecr.api, com.amazonaws.<region>.ecr.dkr, com.amazonaws.<region>.s3)

SHOULD:
- For ECR cross-region: use ECR replication or pull-through cache
- For ECR cross-account: configure ECR repository policy to allow the cluster account
- Use `imagePullPolicy: IfNotPresent` to reduce pull frequency (but not for `latest` tag)

MAY:
- Set up ECR pull-through cache for public registries (Docker Hub, Quay, etc.)
- Use ECR image scanning to verify images before deployment

## Common Issues

- symptoms: "Failed to pull image: repository does not exist or may require 'docker login'"
  diagnosis: "Image name is wrong or the registry requires authentication."
  resolution: "Verify image name. For private registries, add imagePullSecrets."

- symptoms: "Failed to pull image from ECR: no basic auth credentials"
  diagnosis: "Node IAM role lacks ECR permissions."
  resolution: "Attach AmazonEC2ContainerRegistryReadOnly policy to the node role."

- symptoms: "Failed to pull image: dial tcp: i/o timeout"
  diagnosis: "Node cannot reach the container registry — network issue."
  resolution: "Check NAT gateway, VPC endpoints, security groups, and NACLs."

- symptoms: "Failed to pull image from cross-account ECR"
  diagnosis: "ECR repository policy doesn't allow access from the cluster's account."
  resolution: "Add a repository policy allowing the cluster account's node role."

## Output Format

```yaml
root_cause: "Image pull error — <specific_cause>"
evidence:
  - type: pod_events
    content: "<image pull error message>"
  - type: image_config
    content: "<image name, tag, registry>"
severity: HIGH
mitigation:
  immediate: "Fix image reference, credentials, or network connectivity"
  long_term: "Use ECR with proper IAM roles, VPC endpoints for private subnets"
```

## Safety Ratings
GREEN — Triage uses read-only commands (`kubectl describe pod`, `aws ecr describe-images`). Remediation involves fixing image references, creating imagePullSecrets, or adjusting IAM policies — changes scoped to the affected workload with no cluster-wide impact.

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
