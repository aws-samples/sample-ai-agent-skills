---
title: "F1 â€” Pull Throughput Issues"
description: "Diagnose ECR pull throughput bottlenecks including rate limits, network, and VPC endpoint issues"
status: active
severity: MEDIUM
triggers:
  - "slow pull"
  - "pull rate limit"
  - "429 too many requests"
  - "ecr throughput"
  - "pull performance"
owner: devops-agent
objective: "Resolve pull throughput bottlenecks and optimize image delivery"
context: "ECR pull throughput can be limited by API rate limits (1000 pulls/sec/region), network bandwidth (NAT gateway, VPC endpoints), image size, and concurrent pull volume. Large-scale deployments (EKS, ECS) can hit these limits."
---

## Phase 1 â€” Triage

MUST:
- Check for 429 errors in application/container runtime logs
- Verify VPC endpoint exists for ECR: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.ecr.dkr`
- Check NAT gateway throughput if not using VPC endpoints
- Monitor CloudWatch metrics for pull counts

SHOULD:
- Check image sizes: `aws ecr describe-images --repository-name <repo> --query 'imageDetails[*].{tag:imageTags[0],size:imageSizeInBytes}'`
- Verify S3 gateway endpoint exists (ECR layers stored in S3): `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.s3`
- Check concurrent pull volume during deployments

MAY:
- Profile network latency to ECR endpoints
- Check if multi-architecture images are pulling unnecessary platforms

## Phase 2 â€” Remediate

MUST:
- If rate limited, implement exponential backoff in container runtime
- Create VPC endpoints for ECR (`ecr.dkr` and `ecr.api`) and S3 gateway endpoint
- Ensure security groups on VPC endpoints allow HTTPS (443)

SHOULD:
- Optimize image sizes (multi-stage builds, smaller base images)
- Use pull-through cache for frequently accessed upstream images
- Stagger deployments to avoid pull storms

## Output Format

```yaml
root_cause: "pull_throughput â€” <specific_cause>"
evidence:
  - type: rate_limit_errors
    content: "<429 error details>"
  - type: network_config
    content: "<VPC endpoint and NAT config>"
severity: MEDIUM
mitigation:
  immediate: "Add VPC endpoints or implement backoff"
  long_term: "Optimize image sizes and use pull-through caching"
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
