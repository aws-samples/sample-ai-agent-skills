---
title: "E2 â€” Encryption Issues"
description: "Diagnose ECR encryption problems including KMS key access, cross-account encryption, and key rotation"
status: active
severity: HIGH
triggers:
  - "ecr encryption"
  - "kms ecr"
  - "encryption error ecr"
  - "cannot decrypt image"
  - "KMS key ecr"
owner: devops-agent
objective: "Resolve ECR encryption issues and ensure proper KMS key access"
context: "ECR supports AES-256 (default) and KMS encryption. KMS encryption requires the pushing/pulling principal to have KMS permissions. Cross-account access with KMS encryption requires key policy grants to the remote account."
---

## Phase 1 â€” Triage

MUST:
- Check repository encryption config: `aws ecr describe-repositories --repository-names <repo> --query 'repositories[0].encryptionConfiguration'`
- If KMS, verify key exists and is enabled: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify caller has `kms:Decrypt` and `kms:GenerateDataKey` permissions

SHOULD:
- Check if KMS key is in the same region as the repository
- Verify no KMS key policy deny statements blocking access
- Check CloudTrail for KMS access denied events

MAY:
- Check KMS key grants: `aws kms list-grants --key-id <key-id>`
- Verify key rotation status: `aws kms get-key-rotation-status --key-id <key-id>`

## Phase 2 â€” Remediate

MUST:
- Grant KMS permissions to the principal: `kms:Decrypt`, `kms:GenerateDataKey`, `kms:DescribeKey`
- For cross-account, add remote account to KMS key policy
- If key is disabled, re-enable: `aws kms enable-key --key-id <key-id>`

SHOULD:
- Use KMS key aliases for easier management
- Enable automatic key rotation for compliance

## Output Format

```yaml
root_cause: "encryption_issue â€” <specific_cause>"
evidence:
  - type: encryption_config
    content: "<repository encryption configuration>"
  - type: kms_key_status
    content: "<KMS key details>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key permissions or key status"
  long_term: "Use IaC to manage KMS key policies alongside ECR repositories"
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
