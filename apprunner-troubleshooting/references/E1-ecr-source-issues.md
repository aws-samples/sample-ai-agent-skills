---
title: "E1 — App Runner ECR Source Issues"
description: "Diagnose ECR image source configuration and automatic deployment problems"
status: active
severity: MEDIUM
triggers:
  - "ECR source"
  - "ECR image"
  - "image repository"
  - "automatic deployment ECR"
  - "image tag"
  - "ECR configuration"
owner: devops-agent
objective: "Identify and fix ECR source configuration issues for App Runner services"
context: "ECR image sources use a pre-built container image from Amazon ECR. The service can be configured for manual or automatic deployments (triggered on image push). Common issues include incorrect image URI, tag mutability problems, cross-account access, automatic deployment not triggering, and access role permissions."
---

## Phase 1 — Triage

MUST:
- Check image source configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.ImageRepository'`
- Check authentication configuration (access role): `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.AuthenticationConfiguration'`
- Check auto-deployment setting: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.AutoDeploymentsEnabled'`
- Verify the ECR image exists and is accessible: `aws ecr describe-images --repository-name <repo-name> --image-ids imageTag=<tag>`
- List recent operations to check deployment history: `aws apprunner list-operations --service-arn <service-arn>`

SHOULD:
- Check ECR repository tag mutability: `aws ecr describe-repositories --repository-names <repo-name> --query 'repositories[*].imageTagMutability'`
- Verify the access role has correct ECR permissions
- Check if the ECR repository is in the same region as the App Runner service

MAY:
- Check ECR lifecycle policies that might delete images: `aws ecr get-lifecycle-policy --repository-name <repo-name>`
- Verify ECR image scan results for vulnerabilities: `aws ecr describe-image-scan-findings --repository-name <repo-name> --image-id imageTag=<tag>`

## Phase 2 — Remediate

MUST:
- Ensure the image URI format is correct: `<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`
- Verify the access role has ECR permissions and correct trust policy (build.apprunner.amazonaws.com)
- For automatic deployments, ensure AutoDeploymentsEnabled is true and the image tag is being pushed to

SHOULD:
- Use immutable tags for production to prevent accidental overwrites
- Use the `latest` tag or a specific mutable tag for automatic deployments
- Ensure ECR lifecycle policies do not delete images that are actively referenced

MAY:
- Set up ECR image scanning for security compliance
- Use ECR replication for cross-region redundancy

## Common Issues

- symptoms: "Automatic deployment not triggering when new image is pushed"
  diagnosis: "AutoDeploymentsEnabled is false, or the image is pushed to a different tag than configured."
  resolution: "Enable auto-deployments and ensure the image is pushed to the exact tag configured in the service."

- symptoms: "Service uses an outdated image despite pushing a new one"
  diagnosis: "Image tag is immutable and the new push created a different tag, or auto-deployment is disabled."
  resolution: "Use a mutable tag (e.g., latest) with auto-deployment, or trigger manual deployment."

- symptoms: "Cross-account ECR image pull fails"
  diagnosis: "ECR repository policy does not grant access to the App Runner service account."
  resolution: "Add a repository policy granting ecr:BatchGetImage and ecr:GetDownloadUrlForLayer to the App Runner account."

## Output Format

```yaml
root_cause: "ecr_source — <specific_cause>"
evidence:
  - type: image_config
    content: "<image URI, tag, auto-deployment setting>"
  - type: access_role
    content: "<role ARN and permissions>"
  - type: ecr_repository
    content: "<repository details and tag mutability>"
severity: MEDIUM
mitigation:
  immediate: "Fix ECR configuration and trigger deployment"
  long_term: "Implement image tagging strategy and lifecycle policies"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure image URI format is correct | GREEN | Configuration fix |
| Verify access role permissions | GREEN | Read-only check |
| Enable AutoDeploymentsEnabled | YELLOW | Changes deployment trigger behavior |
| Use immutable tags for production | GREEN | Best practice |
| Set up ECR image scanning | GREEN | Security improvement |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Image configuration**: ECR image URI, tag, auto-deployment — INTERNAL
- **Access role**: IAM role ARN and permissions — INTERNAL
- **ECR repository**: Repository details and tag mutability — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER enable auto-deployment on a production service without testing the pipeline

## Phase 3 — Rollback

1. Revert image URI to previous known-good image tag
2. Disable auto-deployment if it was enabled and caused issues
3. Trigger manual deployment with the previous image
4. Revert access role changes if permissions were modified
5. Verify service is healthy with the rolled-back image

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
