---
title: "B3 â€” Tag Immutability Issues"
description: "Diagnose ECR tag immutability conflicts including push failures and workflow impacts"
status: active
severity: MEDIUM
triggers:
  - "tag immutability"
  - "ImageTagAlreadyExistsException"
  - "cannot overwrite tag"
  - "immutable tag"
  - "tag already exists"
owner: devops-agent
objective: "Resolve tag immutability conflicts while maintaining image integrity"
context: "Tag immutability prevents overwriting existing image tags. When enabled, pushing an image with an existing tag fails. This is a repository-level setting affecting all tags. Common in CI/CD pipelines using mutable tags like 'latest'."
---

## Phase 1 â€” Triage

MUST:
- Check immutability setting: `aws ecr describe-repositories --repository-names <repo> --query 'repositories[0].imageTagMutability'`
- Verify the conflicting tag exists: `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>`
- Check if the push is using a tag that already exists

SHOULD:
- Review CI/CD pipeline tagging strategy
- Check if other repositories have different immutability settings

MAY:
- List all tags in the repository: `aws ecr describe-images --repository-name <repo> --query 'imageDetails[*].imageTags'`

## Phase 2 â€” Remediate

MUST:
- If mutable tags needed (e.g., `latest`), disable immutability: `aws ecr put-image-tag-mutability --repository-name <repo> --image-tag-mutability MUTABLE`
- If immutability desired, switch to unique tags (git SHA, build number, timestamp)

SHOULD:
- Use a tagging strategy that combines unique tags with mutable convenience tags
- Document the immutability policy for the team

## Common Issues

- symptoms: "ImageTagAlreadyExistsException on push"
  diagnosis: "Tag immutability enabled and tag already exists."
  resolution: "Use unique tag or disable immutability."

- symptoms: "CI/CD pipeline fails pushing 'latest'"
  diagnosis: "Immutability prevents overwriting 'latest' tag."
  resolution: "Disable immutability or change pipeline to use unique tags."

## Output Format

```yaml
root_cause: "tag_immutability â€” <specific_cause>"
evidence:
  - type: repository_config
    content: "<imageTagMutability setting>"
severity: MEDIUM
mitigation:
  immediate: "Adjust immutability setting or tagging strategy"
  long_term: "Implement unique tagging strategy (git SHA + convenience tags)"
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
