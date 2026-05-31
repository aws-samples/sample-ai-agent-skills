---
title: "D2 â€” Cross-Account Replication"
description: "Diagnose ECR cross-account replication failures including permissions and registry policies"
status: active
severity: MEDIUM
triggers:
  - "cross-account replication"
  - "replicate to another account"
  - "replication permission denied"
  - "registry policy replication"
owner: devops-agent
objective: "Resolve cross-account replication issues and ensure images replicate to target accounts"
context: "Cross-account replication requires the destination account to grant replication permissions via a registry policy. The source account configures replication rules targeting the destination. Both accounts must be properly configured."
---

## Phase 1 â€” Triage

MUST:
- Check source replication config: `aws ecr describe-registry --query 'replicationConfiguration'`
- In destination account, check registry policy: `aws ecr get-registry-policy`
- Verify destination registry policy allows `ecr:ReplicateImage` and `ecr:CreateRepository` from source account
- Check replication status: `aws ecr describe-image-replication-status --repository-name <repo> --image-id imageTag=<tag>`

SHOULD:
- Verify both accounts are in the same AWS Organization if using org-based policies
- Check CloudTrail in both accounts for replication errors

MAY:
- Verify no SCP blocking cross-account ECR operations
- Check service quotas in destination account

## Phase 2 â€” Remediate

MUST:
- In destination account, set registry policy:
  ```
  aws ecr put-registry-policy --policy-text '{
    "Version":"2012-10-17",
    "Statement":[{
      "Sid":"AllowReplication",
      "Effect":"Allow",
      "Principal":{"AWS":"arn:aws:iam::<source-account>:root"},
      "Action":["ecr:ReplicateImage","ecr:CreateRepository"]
    }]
  }'
  ```
- In source account, configure replication targeting destination account

SHOULD:
- Use Organization ID condition for broader org-level replication
- Configure lifecycle policies independently in destination account

## Output Format

```yaml
root_cause: "cross_account_replication â€” <specific_cause>"
evidence:
  - type: source_replication_config
    content: "<source replication rules>"
  - type: destination_registry_policy
    content: "<destination registry policy>"
severity: MEDIUM
mitigation:
  immediate: "Configure registry policy in destination and replication rules in source"
  long_term: "Use Organization-based policies for multi-account replication"
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
