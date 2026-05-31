---
title: "B2 â€” Lifecycle Policy Issues"
description: "Diagnose ECR lifecycle policy problems including rules not executing, wrong images deleted, and policy conflicts"
status: active
severity: MEDIUM
triggers:
  - "lifecycle policy not working"
  - "images not cleaned up"
  - "wrong images deleted"
  - "lifecycle rule"
  - "ecr cleanup"
owner: devops-agent
objective: "Fix lifecycle policy configuration to correctly manage image retention"
context: "ECR lifecycle policies automate image cleanup based on age, count, or tag status. Policies evaluate asynchronously within 24 hours. Common issues include rule priority conflicts, incorrect tag filters, and unexpected image deletion."
---

## Phase 1 â€” Triage

MUST:
- Get current lifecycle policy: `aws ecr get-lifecycle-policy --repository-name <repo>`
- Preview policy effect: `aws ecr get-lifecycle-policy-preview --repository-name <repo>`
- List current images: `aws ecr describe-images --repository-name <repo> --query 'imageDetails[*].{tags:imageTags,pushed:imagePushedAt,digest:imageDigest}'`
- Check CloudTrail for lifecycle deletions: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=BatchDeleteImage --max-results 10`

SHOULD:
- Verify rule priorities (lower number = higher priority, evaluated first)
- Check if rules target tagged vs untagged images correctly
- Verify `countType` (imageCountMoreThan vs sinceImagePushed)

MAY:
- Start a lifecycle policy preview: `aws ecr start-lifecycle-policy-preview --repository-name <repo> --lifecycle-policy-text file://policy.json`
- Compare policy with image inventory to predict deletions

## Phase 2 â€” Remediate

MUST:
- Fix rule priorities if conflicting (each rule needs unique priority)
- Correct tag prefix filters to match actual image tags
- Use lifecycle policy preview to validate before applying

SHOULD:
- Always keep a minimum number of images with `countNumber`
- Use `tagPrefixList` to protect specific tag patterns
- Test with preview before applying to production repositories

## Common Issues

- symptoms: "lifecycle policy exists but images not being cleaned"
  diagnosis: "Policy evaluation is asynchronous, up to 24 hours."
  resolution: "Wait 24 hours. Verify rules match target images with preview."

- symptoms: "important images were deleted"
  diagnosis: "Rule priority or tag filter too broad."
  resolution: "Add higher-priority rule to protect important tags. Use preview to validate."

## Output Format

```yaml
root_cause: "lifecycle_policy_issue â€” <specific_cause>"
evidence:
  - type: policy_config
    content: "<lifecycle policy JSON>"
  - type: preview_result
    content: "<lifecycle policy preview>"
severity: MEDIUM
mitigation:
  immediate: "Fix lifecycle policy rules and priorities"
  long_term: "Use policy preview in CI/CD before applying changes"
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
