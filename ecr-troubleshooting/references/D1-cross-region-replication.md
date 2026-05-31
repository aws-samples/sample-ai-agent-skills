---
title: "D1 â€” Cross-Region Replication"
description: "Diagnose ECR cross-region replication failures including configuration, lag, and filter issues"
status: active
severity: MEDIUM
triggers:
  - "cross-region replication"
  - "replication not working"
  - "image not replicated"
  - "replication lag"
  - "ecr replication"
owner: devops-agent
objective: "Resolve cross-region replication issues and ensure images are available in target regions"
context: "ECR replication copies images to other regions or accounts. Replication is configured at the registry level. Only image data is replicated â€” policies, scanning config, and settings are NOT replicated."
---

## Phase 1 â€” Triage

MUST:
- Check replication configuration: `aws ecr describe-registry --query 'replicationConfiguration'`
- Verify replication rules include the target region
- Check replication status for specific image: `aws ecr describe-image-replication-status --repository-name <repo> --image-id imageTag=<tag>`
- Verify the repository exists in the destination region: `aws ecr describe-repositories --repository-names <repo> --region <target-region>`

SHOULD:
- Check if replication filters are excluding the image (prefix filters)
- Verify IAM permissions for replication: `ecr:ReplicateImage` in destination
- Check CloudTrail for replication events

MAY:
- Check CloudWatch metrics for replication in both regions
- Verify no service quota limits in destination region

## Phase 2 â€” Remediate

MUST:
- If replication not configured, set it up: `aws ecr put-replication-configuration --replication-configuration '{"rules":[{"destinations":[{"region":"<target-region>","registryId":"<account>"}]}]}'`
- If filter excluding images, adjust prefix filter or remove filter
- If destination repo missing, it will be auto-created on first replication

SHOULD:
- Configure lifecycle policies separately in destination region
- Set up monitoring for replication status

## Output Format

```yaml
root_cause: "cross_region_replication â€” <specific_cause>"
evidence:
  - type: replication_config
    content: "<replication configuration>"
  - type: replication_status
    content: "<image replication status>"
severity: MEDIUM
mitigation:
  immediate: "Fix replication configuration or filters"
  long_term: "Monitor replication status and configure destination policies"
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
