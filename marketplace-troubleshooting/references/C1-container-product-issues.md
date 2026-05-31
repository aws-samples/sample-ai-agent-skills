---
title: "C1 — Container Product Issues"
description: "Diagnose container-based Marketplace product deployment issues"
status: active
severity: HIGH
triggers: ["container product failed", "ecs marketplace", "eks marketplace", "container image pull error"]
owner: devops-agent
objective: "Resolve container product deployment and access issues"
context: "Container products provide ECR images. Deployment requires existing ECS/EKS infrastructure and proper IAM permissions for image pull."
---

## Phase 1 — Triage

MUST:
- Verify subscription and entitlements: `aws marketplace-entitlement get-entitlements --product-code <product-code>`
- Check ECR image access: `aws ecr describe-images --registry-id <seller-account-id> --repository-name <repo-name>`
- Verify ECS/EKS cluster status: `aws ecs describe-clusters --clusters <cluster-name>` or `aws eks describe-cluster --name <cluster-name>`

SHOULD:
- Check task/pod logs for image pull errors: `aws ecs describe-tasks --cluster <cluster> --tasks <task-arn>`
- Verify IAM task execution role has ECR pull permissions

MAY:
- Check network connectivity to ECR endpoints
- Review VPC endpoint configuration for ECR

## Phase 2 — Remediate

MUST:
- Ensure active Marketplace subscription before pulling images
- Configure IAM task execution role with ecr:GetDownloadUrlForLayer, ecr:BatchGetImage permissions
- Verify cluster has sufficient capacity for the container workload

SHOULD:
- Use VPC endpoints for ECR in private subnets: `aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.<region>.ecr.dkr"`
- Configure proper resource limits in task definitions

MAY:
- Set up container health checks
- Configure auto-scaling for the container service

## Safety Ratings

```
safety_ratings:
  - "marketplace-entitlement get-entitlements: GREEN — read-only check"
  - "ecr describe-images: GREEN — read-only image listing"
  - "ecs describe-clusters: GREEN — read-only cluster status"
  - "ecs create-service: RED — deploys containers with potential cost impact"
  - "ec2 describe-vpc-endpoints: GREEN — read-only network check"
```

## Escalation Conditions

- Container image pull fails with authentication errors after IAM fixes
- Container requires specific instance types not available in the cluster
- Seller's container image has known vulnerabilities
- Container product requires persistent storage not configured

## Data Sensitivity

- `aws ecr describe-images`: LOW — returns image metadata
- `aws ecs describe-tasks`: MEDIUM — reveals container configuration and environment variables
- Container environment variables: HIGH — may contain secrets and API keys
- Container logs: HIGH — may contain application data

## Prohibited Actions

- NEVER expose container ports to 0.0.0.0/0 without explicit approval
- NEVER run containers with privileged mode unless required by the product
- NEVER store secrets in container environment variables — use Secrets Manager
- NEVER deploy to production clusters without testing in staging first

## Phase 3 — Rollback

1. Stop failed ECS services: `aws ecs update-service --cluster <cluster> --service <service> --desired-count 0`
2. Delete failed task definitions if not needed: `aws ecs deregister-task-definition --task-definition <task-def>`
3. For EKS, delete failed deployments: `kubectl delete deployment <name>`
4. Remove VPC endpoints created for troubleshooting if not needed
5. Verify no orphaned load balancers or target groups remain

## Common Issues

- symptoms: "ECS task fails with CannotPullContainerError"
  diagnosis: "IAM task execution role missing ECR permissions or subscription not active."
  resolution: "Verify Marketplace subscription. Add ecr:GetDownloadUrlForLayer and ecr:BatchGetImage to task execution role."

## Output Format

```yaml
root_cause: "container_product — <specific_cause>"
evidence:
  - type: container_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix container deployment issue"
  long_term: "Use IaC for container deployments with proper IAM"
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
