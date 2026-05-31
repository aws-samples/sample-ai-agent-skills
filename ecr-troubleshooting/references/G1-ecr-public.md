---
title: "G1 â€” ECR Public Issues"
description: "Diagnose ECR Public repository problems including publishing, authentication, and rate limits"
status: active
severity: MEDIUM
triggers:
  - "ecr public"
  - "public.ecr.aws"
  - "public repository"
  - "ecr public push"
  - "public registry"
owner: devops-agent
objective: "Resolve ECR Public repository issues and ensure public image availability"
context: "ECR Public is a separate service from private ECR with its own API (ecr-public), authentication (ecr-public get-login-password), and endpoint (public.ecr.aws). It has different rate limits and requires us-east-1 for management operations."
---

## Phase 1 â€” Triage

MUST:
- Verify using correct CLI commands: `aws ecr-public` (NOT `aws ecr`)
- Check authentication: `aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws`
- List public repositories: `aws ecr-public describe-repositories --region us-east-1`
- Verify management operations use us-east-1 region

SHOULD:
- Check repository catalog data: `aws ecr-public get-repository-catalog-data --repository-name <repo> --region us-east-1`
- Verify IAM permissions include `ecr-public:*` actions
- Check for rate limiting on unauthenticated pulls

MAY:
- Check ECR Public gallery listing: `aws ecr-public describe-registries --region us-east-1`
- Verify registry alias: `aws ecr-public describe-registries --region us-east-1 --query 'registries[0].aliases'`

## Phase 2 â€” Remediate

MUST:
- Use `ecr-public` CLI commands, not `ecr` commands
- Authenticate to `public.ecr.aws` for push operations
- Ensure all management API calls go to us-east-1

SHOULD:
- Authenticate pulls to get higher rate limits
- Configure catalog data for gallery visibility

## Output Format

```yaml
root_cause: "ecr_public â€” <specific_cause>"
evidence:
  - type: repository_status
    content: "<ecr-public describe-repositories output>"
  - type: auth_status
    content: "<authentication result>"
severity: MEDIUM
mitigation:
  immediate: "Use correct ecr-public commands and us-east-1 region"
  long_term: "Automate ECR Public publishing in CI/CD"
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
