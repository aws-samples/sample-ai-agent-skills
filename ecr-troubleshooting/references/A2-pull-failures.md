---
title: "A2 â€” Pull Failures"
description: "Diagnose ECR Docker pull failures including manifest not found, access denied, and rate limits"
status: active
severity: HIGH
triggers:
  - "docker pull failed"
  - "manifest not found"
  - "pull access denied"
  - "image not found ecr"
  - "pull error"
owner: devops-agent
objective: "Identify why Docker pull from ECR is failing and restore pull capability"
context: "ECR pull failures commonly stem from expired auth tokens, wrong image tag/digest, missing IAM permissions, repository policy restrictions, rate limiting, or network connectivity issues through VPC endpoints."
---

## Phase 1 â€” Triage

MUST:
- Verify auth token is current: `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com`
- Verify image exists in repository: `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>`
- Check IAM permissions include `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:BatchCheckLayerAvailability`
- If cross-account, check repository policy: `aws ecr get-repository-policy --repository-name <repo>`
- Check for rate limiting in CloudWatch: look for 429 errors in application logs

SHOULD:
- Verify full image URI is correct: `<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`
- Check VPC endpoint configuration if pulling from within VPC: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.ecr.dkr`
- Verify security group on VPC endpoint allows HTTPS (443)

MAY:
- Check if image was deleted by lifecycle policy: `aws ecr get-lifecycle-policy-preview --repository-name <repo>`
- Test with image digest instead of tag: `docker pull <uri>@sha256:<digest>`

## Phase 2 â€” Remediate

MUST:
- If auth expired, re-authenticate with `get-login-password`
- If image not found, verify correct repository, tag, and region
- If access denied cross-account, add repository policy granting `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`
- If rate limited, implement exponential backoff and consider VPC endpoints

SHOULD:
- Configure VPC endpoints for ECR (both `ecr.dkr` and `ecr.api`) for private subnet access
- Ensure S3 gateway endpoint exists for layer downloads

## Common Issues

- symptoms: "manifest for <image>:<tag> not found"
  diagnosis: "Image tag does not exist or was deleted by lifecycle policy."
  resolution: "Verify tag exists with describe-images. Check lifecycle policy history."

- symptoms: "pull access denied or repository does not exist"
  diagnosis: "Missing IAM permissions, wrong account/region, or repository policy blocking."
  resolution: "Verify IAM policy, repository policy, and correct registry URI."

- symptoms: "429 Too Many Requests"
  diagnosis: "ECR pull rate limit exceeded."
  resolution: "Implement backoff. Use VPC endpoints. Consider pull-through cache."

## Output Format

```yaml
root_cause: "pull_failure â€” <specific_cause>"
evidence:
  - type: image_status
    content: "<describe-images output>"
  - type: auth_status
    content: "<login result>"
severity: HIGH
mitigation:
  immediate: "Fix authentication, permissions, or image reference"
  long_term: "Configure VPC endpoints and pull-through caching"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — ECR hosts container images critical to deployments |
| Data Risk | MEDIUM — Images may contain application code and secrets |
| Recovery Complexity | MEDIUM — Deleted images can be rebuilt but cause deployment delays |
| Blast Radius | HIGH — Image issues affect all services pulling from the repository |

## Escalation Conditions

- Image pull failures affecting production deployments or auto-scaling events
- Repository policy changes locking out CI/CD pipelines from pushing images
- Cross-account replication failures causing stale images in secondary regions
- Image scanning revealing critical CVEs in production-running images
- Lifecycle policy unexpectedly deleting images referenced by running tasks
- Encryption key issues preventing image push or pull operations

## Data Sensitivity

- Container images may embed application secrets, API keys, or credentials
- Repository policies may reveal cross-account trust relationships
- Image tags and digests are deployment identifiers — treat as operational metadata
- Scan results may expose vulnerability details that should not be shared broadly
- ECR auth tokens grant broad repository access — treat as short-lived secrets

## PROHIBITED Actions

- NEVER suggest deleting images that are referenced by running tasks or services
- NEVER modify repository policies without verifying CI/CD pipeline access requirements
- NEVER disable tag immutability on repositories enforcing deployment traceability
- NEVER delete a repository without confirming all dependent services have been migrated
- NEVER share or log ECR authentication tokens in plain text
- NEVER disable image scanning on repositories subject to compliance requirements

## Phase 3 — Rollback

- If image was accidentally deleted: rebuild and push from CI/CD pipeline or pull from replication target
- If repository policy change blocks access: revert to previous policy JSON from version control
- If lifecycle policy deleted needed images: restore from cross-region replica or rebuild from source
- If tag immutability was disabled: re-enable and audit for any overwritten tags
- If encryption configuration was changed: verify KMS key access and re-push affected images
- Document all rollback actions with image digests and tags for audit trail
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
