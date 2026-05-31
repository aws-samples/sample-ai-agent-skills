---
title: "C1 â€” Repository Creation Failures"
description: "Diagnose ECR repository creation failures including naming, limits, and encryption issues"
status: active
severity: MEDIUM
triggers:
  - "create repository failed"
  - "repository creation error"
  - "RepositoryAlreadyExistsException"
  - "ecr create-repository"
owner: devops-agent
objective: "Resolve repository creation failures and establish correct configuration"
context: "ECR repository creation can fail due to naming conflicts, service quota limits, IAM permissions, KMS key issues, or invalid configuration parameters."
---

## Phase 1 â€” Triage

MUST:
- Check if repository already exists: `aws ecr describe-repositories --repository-names <repo>`
- Verify IAM permissions include `ecr:CreateRepository`
- Check service quota for repositories: `aws service-quotas get-service-quota --service-code ecr --quota-code L-CFEB8E4D`
- Count current repositories: `aws ecr describe-repositories --query 'repositories | length(@)'`

SHOULD:
- Verify repository name follows naming rules (lowercase, alphanumeric, hyphens, underscores, forward slashes)
- If using KMS encryption, verify key exists and permissions: `aws kms describe-key --key-id <key-id>`
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateRepository --max-results 5`

MAY:
- Check for SCP restrictions on ECR operations
- Verify no tag policy conflicts

## Phase 2 â€” Remediate

MUST:
- If name conflict, use a different name or check if existing repo can be reused
- If quota exceeded, request increase: `aws service-quotas request-service-quota-increase --service-code ecr --quota-code L-CFEB8E4D --desired-value <new-limit>`
- If KMS error, grant `kms:CreateGrant`, `kms:DescribeKey` to ECR service principal

SHOULD:
- Use consistent naming conventions (e.g., `<team>/<service>/<component>`)
- Configure encryption, scanning, and immutability at creation time

## Output Format

```yaml
root_cause: "repository_creation_failure â€” <specific_cause>"
evidence:
  - type: error_message
    content: "<creation error>"
  - type: quota_status
    content: "<current vs max repositories>"
severity: MEDIUM
mitigation:
  immediate: "Fix naming, permissions, or quota"
  long_term: "Automate repository creation with IaC"
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
