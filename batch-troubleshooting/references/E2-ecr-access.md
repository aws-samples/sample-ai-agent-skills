---
title: "E2 — ECR Access Issues"
description: "Diagnose ECR image pull failures in Batch jobs"
status: active
severity: HIGH
triggers:
  - "image pull failed"
  - "ECR access denied"
  - "repository not found"
  - "CannotPullContainerError"
  - "image not found"
owner: devops-agent
objective: "Identify and resolve ECR image pull failures"
context: "ECR access issues include: IAM permissions missing, network access to ECR blocked, image tag not found, cross-account ECR access, or ECR endpoint not reachable."
---

## Phase 1 — Triage

MUST:
- Check job failure reason for image pull errors: `aws batch describe-jobs --jobs <job-id> --query 'jobs[*].container.reason'`
- Verify image exists in ECR: `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>`
- Check execution role has ECR permissions: `aws iam list-attached-role-policies --role-name <execution-role>`
- Verify network access to ECR (VPC endpoint or NAT Gateway)

SHOULD:
- Check ECR repository policy for cross-account access
- Verify ECR endpoint is reachable from CE subnets
- Check for image tag immutability issues

MAY:
- Check ECR lifecycle policies for image deletion
- Verify Docker Hub rate limits if using public images

## Phase 2 — Remediate

MUST:
- Add ECR permissions to execution role: ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:GetDownloadUrlForLayer
- Ensure network path to ECR (VPC endpoint or NAT Gateway)
- Verify image URI is correct: `<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`

SHOULD:
- Create VPC endpoint for ECR: `com.amazonaws.<region>.ecr.dkr` and `com.amazonaws.<region>.ecr.api`
- Create S3 VPC endpoint for ECR layer storage

MAY:
- Implement ECR image scanning
- Set up cross-account ECR access with repository policies

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add ECR permissions to execution role | YELLOW | Permission change — review scope |
| Ensure network path to ECR | GREEN | Network verification — non-destructive |
| Create VPC endpoints for ECR | GREEN | Adds network path — non-destructive |
| Fix image URI in job definition | GREEN | Configuration fix — non-destructive |
| Implement ECR image scanning | GREEN | Security improvement — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- ECR access failure blocks all job executions
- Cross-account ECR access requires repository policy changes
- Image deletion by lifecycle policy causes job failures

## Data Sensitivity

- **Classification: MEDIUM**
- ECR repository names reveal application architecture
- Image tags and versions expose deployment patterns
- Execution role permissions reveal infrastructure access
- Cross-account ECR access reveals organizational trust relationships

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest making ECR repositories publicly accessible
- **NEVER** suggest granting broad ECR permissions (ecr:*) to resolve access issues
- **NEVER** suggest disabling ECR image scanning for production images

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Added ECR permissions to execution role | Remove added ECR permissions if not needed |
| Created VPC endpoints for ECR | Delete VPC endpoints if not needed |
| Updated ECR repository policy | Revert repository policy to previous version |

## Output Format

```yaml
root_cause: "ecr_access_issue — <specific_cause>"
evidence:
  - type: pull_error
    content: "<image pull error message>"
  - type: iam_permissions
    content: "<execution role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or network access to ECR"
  long_term: "Implement VPC endpoints and proper IAM roles"
```

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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"
