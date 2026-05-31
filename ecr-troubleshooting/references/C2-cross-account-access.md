---
title: "C2 â€” Cross-Account Access"
description: "Diagnose ECR cross-account access issues including repository policies and IAM configuration"
status: active
severity: HIGH
triggers:
  - "cross-account ecr"
  - "access denied other account"
  - "cross-account pull"
  - "share ecr repository"
  - "ecr cross-account"
owner: devops-agent
objective: "Enable secure cross-account ECR access with proper IAM and repository policies"
context: "Cross-account ECR access requires BOTH an IAM policy in the pulling account AND a repository resource policy in the ECR account. Missing either results in access denied. KMS-encrypted repos need additional key policy grants."
---

## Phase 1 â€” Triage

MUST:
- Check repository policy: `aws ecr get-repository-policy --repository-name <repo>`
- Verify the remote account/principal is listed in the repository policy
- In the remote account, verify IAM policy includes ECR pull permissions
- If KMS encrypted, check KMS key policy grants access to remote account: `aws kms get-key-policy --key-id <key-id> --policy-name default`

SHOULD:
- Verify the correct registry URI is used: `<source-account>.dkr.ecr.<region>.amazonaws.com/<repo>`
- Check if the remote account can authenticate to the source registry
- Verify no SCP blocking cross-account ECR access

MAY:
- Test with AWS CLI from the remote account: `aws ecr batch-get-image --registry-id <source-account> --repository-name <repo> --image-ids imageTag=<tag>`
- Check CloudTrail in both accounts for denied events

## Phase 2 â€” Remediate

MUST:
- Set repository policy granting cross-account access:
  ```
  aws ecr set-repository-policy --repository-name <repo> --policy-text '{
    "Version":"2012-10-17",
    "Statement":[{
      "Sid":"CrossAccountPull",
      "Effect":"Allow",
      "Principal":{"AWS":"arn:aws:iam::<remote-account>:root"},
      "Action":["ecr:GetDownloadUrlForLayer","ecr:BatchGetImage","ecr:BatchCheckLayerAvailability"]
    }]
  }'
  ```
- In remote account, ensure IAM role has ECR pull permissions
- If KMS encrypted, add remote account to KMS key policy with `kms:Decrypt`

SHOULD:
- Use specific principals instead of account root where possible
- Use AWS Organizations for org-wide repository policies

## Output Format

```yaml
root_cause: "cross_account_access â€” <specific_cause>"
evidence:
  - type: repository_policy
    content: "<repository policy>"
  - type: iam_policy
    content: "<remote account IAM policy>"
severity: HIGH
mitigation:
  immediate: "Configure both repository policy and IAM policy"
  long_term: "Use Organizations-based policies for multi-account access"
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
