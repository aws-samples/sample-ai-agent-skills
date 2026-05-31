---
title: "F2 â€” Caching Issues"
description: "Diagnose ECR pull-through cache problems including rule configuration, upstream auth, and sync failures"
status: active
severity: MEDIUM
triggers:
  - "pull-through cache"
  - "cache rule"
  - "upstream registry"
  - "ecr cache"
  - "cached pull failed"
owner: devops-agent
objective: "Resolve pull-through cache issues and ensure upstream images are cached correctly"
context: "ECR pull-through cache rules automatically cache images from upstream registries (Docker Hub, GitHub, Quay, ECR Public, etc.) on first pull. Common issues include missing credentials, incorrect rule configuration, and upstream registry authentication failures."
---

## Phase 1 â€” Triage

MUST:
- List cache rules: `aws ecr describe-pull-through-cache-rules`
- Verify upstream registry prefix and ECR repository prefix mapping
- Check if credentials are stored in Secrets Manager for authenticated upstreams
- Verify the secret exists and is accessible: `aws secretsmanager describe-secret --secret-id <secret-arn>`

SHOULD:
- Test pulling through the cache: `docker pull <account>.dkr.ecr.<region>.amazonaws.com/<prefix>/<image>:<tag>`
- Check CloudTrail for cache-related errors
- Verify IAM permissions include `ecr:BatchImportUpstreamImage`

MAY:
- Check upstream registry status (Docker Hub, Quay, etc.)
- Verify no network restrictions blocking upstream access

## Phase 2 â€” Remediate

MUST:
- If credentials missing, create Secrets Manager secret and update cache rule
- If rule misconfigured, delete and recreate: `aws ecr create-pull-through-cache-rule --ecr-repository-prefix <prefix> --upstream-registry-url <url> --credential-arn <secret-arn>`
- Ensure IAM role has `secretsmanager:GetSecretValue` for the credential secret

SHOULD:
- Use separate prefixes for different upstream registries
- Monitor cached repository sizes with lifecycle policies

## Output Format

```yaml
root_cause: "caching_issue â€” <specific_cause>"
evidence:
  - type: cache_rules
    content: "<pull-through cache rules>"
  - type: credential_status
    content: "<Secrets Manager secret status>"
severity: MEDIUM
mitigation:
  immediate: "Fix cache rule configuration or credentials"
  long_term: "Monitor cache hit rates and configure lifecycle policies for cached repos"
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
