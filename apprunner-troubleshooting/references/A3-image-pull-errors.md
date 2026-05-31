---
title: "A3 — App Runner Image Pull Errors"
description: "Diagnose ECR image pull failures for App Runner services"
status: active
severity: HIGH
triggers:
  - "image pull"
  - "image not found"
  - "ECR access denied"
  - "repository not found"
  - "manifest not found"
owner: devops-agent
objective: "Identify why App Runner cannot pull the container image from ECR and restore image access"
context: "Image pull errors occur with ECR image sources when App Runner cannot access or find the specified container image. Common causes include incorrect image URI, missing image tag, ECR access role permissions, cross-account ECR access, and ECR repository policies."
---

## Phase 1 — Triage

MUST:
- Verify the image source configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.ImageRepository'`
- Check if the ECR image exists: `aws ecr describe-images --repository-name <repo-name> --image-ids imageTag=<tag>`
- Verify the access role ARN: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.AuthenticationConfiguration'`
- Check the access role permissions: `aws iam get-role-policy --role-name <role-name> --policy-name <policy-name>` or `aws iam list-attached-role-policies --role-name <role-name>`
- Verify the access role trust policy allows apprunner.amazonaws.com: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`

SHOULD:
- Check ECR repository policy for cross-account access: `aws ecr get-repository-policy --repository-name <repo-name>`
- Verify the image URI format: `<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`
- Check if the ECR repository is in the same region as the App Runner service

MAY:
- Test image pull locally: `docker pull <image-uri>`
- Check CloudTrail for ECR access denied events

## Phase 2 — Remediate

MUST:
- Ensure the access role has these ECR permissions: `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchCheckLayerAvailability`, `ecr:DescribeImages`
- Ensure the access role trust policy includes: `{"Service": "build.apprunner.amazonaws.com"}`
- Verify the image URI and tag are correct and the image exists in ECR

SHOULD:
- For cross-account ECR, add a repository policy granting access to the App Runner account
- Use immutable image tags to prevent tag overwrites causing pull failures
- Verify ECR lifecycle policies have not deleted the referenced image

MAY:
- Switch to using image digest instead of tag for deterministic pulls
- Set up ECR replication if the repository is in a different region

## Common Issues

- symptoms: "Access denied when pulling image from ECR"
  diagnosis: "Access role missing ECR permissions or incorrect trust policy."
  resolution: "Add ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:GetDownloadUrlForLayer to the access role. Trust policy must allow build.apprunner.amazonaws.com."

- symptoms: "Image not found or manifest unknown"
  diagnosis: "Image URI or tag is incorrect, or the image was deleted by lifecycle policy."
  resolution: "Verify the image exists: aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>."

- symptoms: "Cross-account image pull fails"
  diagnosis: "ECR repository policy does not grant access to the App Runner account."
  resolution: "Add a repository policy allowing the App Runner account to pull images."

## Output Format

```yaml
root_cause: "image_pull_error — <specific_cause>"
evidence:
  - type: image_config
    content: "<image URI and tag>"
  - type: access_role
    content: "<role ARN and permissions>"
  - type: ecr_image
    content: "<image existence check result>"
severity: HIGH
mitigation:
  immediate: "Fix image URI, permissions, or push the missing image"
  long_term: "Use immutable tags and monitor ECR lifecycle policies"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure access role has ECR permissions | YELLOW | IAM policy change |
| Ensure access role trust policy correct | YELLOW | IAM trust policy change |
| Verify image URI and tag | GREEN | Read-only check |
| Add cross-account repository policy | YELLOW | ECR policy change |
| Use immutable image tags | GREEN | Best practice recommendation |
| Switch to image digest | GREEN | Configuration improvement |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **ECR image URI**: Container image location — INTERNAL
- **Access role ARN**: IAM role details — INTERNAL
- **ECR repository policy**: Cross-account access configuration — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER modify ECR repository policies without understanding cross-account impact

## Phase 3 — Rollback

1. Revert access role IAM policy to previous version
2. Revert ECR repository policy if cross-account access was modified
3. Verify the previous image tag is still available in ECR
4. Trigger redeployment with the known-good image
5. Verify service is healthy after rollback

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
