---
title: "G3 — OpenSearch Snapshot Repository Configuration"
description: "Diagnose and resolve snapshot repository registration and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "repository"
  - "register repository"
  - "snapshot repository"
  - "repository_exception"
  - "S3 repository"
owner: devops-agent
objective: "Successfully register and configure an S3 snapshot repository"
context: "Manual snapshots require registering an S3 repository using the _snapshot API. The registration requires an IAM role with S3 permissions that the OpenSearch domain can assume (via iam:PassRole). The role must trust the OpenSearch service principal. Common issues include incorrect IAM role trust policy, missing S3 permissions, KMS key access, and incorrect PassRole permissions."
---

## Phase 1 — Triage

MUST:
- List existing repositories: `curl -XGET "https://<endpoint>/_snapshot?pretty"`
- Check repository details: `curl -XGET "https://<endpoint>/_snapshot/<repo>?pretty"`
- Verify repository: `curl -XPOST "https://<endpoint>/_snapshot/<repo>/_verify?pretty"`
- Check IAM role: `aws iam get-role --role-name <snapshot-role> --query 'Role.{Arn:Arn,AssumeRolePolicyDocument:AssumeRolePolicyDocument}'`
- Check S3 bucket exists and is accessible: `aws s3 ls s3://<bucket-name>/`

SHOULD:
- Check IAM role policies: `aws iam list-attached-role-policies --role-name <snapshot-role>`
- Check S3 bucket policy for restrictions
- Check KMS key policy if using encryption: `aws kms describe-key --key-id <key-id>`
- Verify iam:PassRole permission for the user/role registering the repository

MAY:
- Check CloudTrail for repository registration attempts
- Check S3 bucket region matches domain region

## Phase 2 — Remediate

MUST:
- Create IAM role with trust policy for OpenSearch: trust principal `es.amazonaws.com`
- Attach S3 permissions to the role: `s3:ListBucket` on bucket ARN, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on bucket ARN/*
- Register repository: `curl -XPUT "https://<endpoint>/_snapshot/<repo>" -H 'Content-Type: application/json' -d '{"type":"s3","settings":{"bucket":"<bucket-name>","region":"<region>","role_arn":"arn:aws:iam::<account>:role/<snapshot-role>"}}'`
- Ensure caller has `iam:PassRole` permission for the snapshot role

SHOULD:
- Add KMS permissions if using encrypted snapshots: `kms:GenerateDataKey`, `kms:Decrypt` on the KMS key
- Use a dedicated S3 bucket for snapshots
- Test with a small snapshot after registration

MAY:
- Configure server-side encryption on the S3 bucket
- Set S3 lifecycle policies to manage old snapshot storage costs
- Use S3 bucket in the same region as the domain for performance

## Common Issues

- symptoms: "repository_exception: Could not determine repository generation"
  diagnosis: "IAM role cannot access the S3 bucket. Permission or trust policy issue."
  resolution: "Verify IAM role trust policy includes es.amazonaws.com. Check S3 permissions."

- symptoms: "Access denied when registering repository"
  diagnosis: "Caller lacks iam:PassRole permission for the snapshot role."
  resolution: "Add iam:PassRole permission for the snapshot role ARN to the caller's IAM policy."

- symptoms: "Repository verification fails"
  diagnosis: "S3 bucket not accessible from the domain. Region mismatch or bucket policy blocking."
  resolution: "Verify bucket exists, is in the correct region, and bucket policy allows access."

## Output Format

```yaml
root_cause: "repository_configuration — <specific_cause>"
evidence:
  - type: repository_config
    content: "<repository settings>"
  - type: iam_role
    content: "<role ARN, trust policy, permissions>"
  - type: s3_bucket
    content: "<bucket name, region, accessibility>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM role permissions and re-register repository"
  long_term: "Document repository setup, automate snapshot scheduling"
```


## Safety Ratings
```
safety_ratings:
  - "List and verify repositories: GREEN — read-only API calls"
  - "Check IAM role and S3 bucket: GREEN — read-only inspection"
  - "Register snapshot repository: YELLOW — creates new repository configuration"
  - "Fix IAM role permissions: YELLOW — changes S3 access scope"
  - "Create IAM role with trust policy: YELLOW — creates new IAM resource"
```

## Escalation Conditions
- Domain serves production search
- Repository registration blocking backup capability
- IAM role trust policy changes requiring security review
- Cross-account S3 access needed
- KMS key access for encrypted snapshots

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "IAM role ARN and trust policy: permission configuration"
    - "S3 bucket name: snapshot storage location"
    - "KMS key ID: encryption key identifier"
  handling: "Do not expose IAM role ARNs, S3 bucket names, or KMS key IDs externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER grant overly broad S3 permissions (s3:*) to the snapshot role
- NEVER use the same S3 bucket for snapshots from untrusted domains

## Phase 3 — Rollback
- If repository was registered: delete the repository registration (snapshots in S3 remain)
- If IAM role was created: delete the role if not needed
- If IAM role permissions were changed: revert to previous policy
- If S3 bucket policy was modified: restore previous bucket policy

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
