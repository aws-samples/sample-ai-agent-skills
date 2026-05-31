---
title: "G2 â€” Gallery Issues"
description: "Diagnose ECR Public Gallery visibility and catalog data problems"
status: active
severity: LOW
triggers:
  - "ecr gallery"
  - "public gallery"
  - "gallery listing"
  - "catalog data"
  - "gallery visibility"
owner: devops-agent
objective: "Resolve ECR Public Gallery visibility and catalog data issues"
context: "ECR Public Gallery (gallery.ecr.aws) displays public repositories with catalog data (descriptions, logos, usage examples). Repositories may not appear in gallery search if catalog data is incomplete or the repository is newly created."
---

## Phase 1 â€” Triage

MUST:
- Check catalog data: `aws ecr-public get-repository-catalog-data --repository-name <repo> --region us-east-1`
- Verify repository exists: `aws ecr-public describe-repositories --repository-names <repo> --region us-east-1`
- Check if repository has at least one pushed image
- Verify registry alias is configured

SHOULD:
- Check if catalog data includes description, about text, and usage text
- Verify logo image is valid and within size limits
- Check operating system and architecture tags

MAY:
- Search gallery for the repository to verify visibility
- Check if repository was recently created (gallery indexing may take time)

## Phase 2 â€” Remediate

MUST:
- Update catalog data with required fields:
  ```
  aws ecr-public put-repository-catalog-data --repository-name <repo> --region us-east-1 --catalog-data '{
    "description":"Short description",
    "aboutText":"Detailed about text",
    "usageText":"Usage instructions",
    "operatingSystems":["Linux"],
    "architectures":["x86-64","ARM 64"]
  }'
  ```
- Ensure at least one image is pushed to the repository

SHOULD:
- Add a logo image for better gallery visibility
- Include comprehensive usage examples in catalog data

## Output Format

```yaml
root_cause: "gallery_issue â€” <specific_cause>"
evidence:
  - type: catalog_data
    content: "<repository catalog data>"
severity: LOW
mitigation:
  immediate: "Update catalog data with required fields"
  long_term: "Automate catalog data updates in CI/CD pipeline"
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
