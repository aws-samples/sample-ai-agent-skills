---
title: "A1 â€” Push Failures"
description: "Diagnose ECR Docker push failures including auth errors, denied access, and size limits"
status: active
severity: HIGH
triggers:
  - "docker push failed"
  - "push denied"
  - "authorization failed"
  - "cannot push to ecr"
  - "push error"
owner: devops-agent
objective: "Identify why Docker push to ECR is failing and restore push capability"
context: "ECR push failures commonly stem from expired auth tokens, missing IAM permissions, tag immutability conflicts, repository not existing, or image size limits. The auth token expires every 12 hours."
---

## Phase 1 â€” Triage

MUST:
- Verify auth token is current: `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com`
- Check repository exists: `aws ecr describe-repositories --repository-names <repo>`
- Verify IAM permissions include `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`, `ecr:BatchCheckLayerAvailability`
- Check tag immutability: `aws ecr describe-repositories --repository-names <repo> --query 'repositories[0].imageTagMutability'`
- If KMS encrypted, verify KMS permissions: `aws ecr describe-repositories --repository-names <repo> --query 'repositories[0].encryptionConfiguration'`

SHOULD:
- Check Docker daemon is running and image exists locally: `docker images <image>`
- Verify image tag format matches repository URI: `<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`
- Check CloudTrail for denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutImage --max-results 10`

MAY:
- Check ECR service quotas: `aws service-quotas get-service-quota --service-code ecr --quota-code L-DB2BD2A5`
- Verify no SCP blocking ECR operations

## Phase 2 â€” Remediate

MUST:
- If auth expired, re-authenticate: `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com`
- If repository missing, create it: `aws ecr create-repository --repository-name <repo>`
- If tag immutability conflict, use a unique tag or disable: `aws ecr put-image-tag-mutability --repository-name <repo> --image-tag-mutability MUTABLE`
- If KMS error, grant `kms:GenerateDataKey` and `kms:Decrypt` to the pushing principal

SHOULD:
- Ensure Docker image is properly tagged with full ECR URI before push
- Verify network connectivity to ECR endpoint (VPC endpoint or internet)

## Common Issues

- symptoms: "no basic auth credentials"
  diagnosis: "Docker auth token expired or never obtained."
  resolution: "Run get-login-password and docker login again."

- symptoms: "denied: Your authorization token has expired"
  diagnosis: "Auth token older than 12 hours."
  resolution: "Refresh token. Tokens are valid for 12 hours only."

- symptoms: "ImageTagAlreadyExistsException"
  diagnosis: "Tag immutability enabled and tag already exists."
  resolution: "Use unique tag or disable tag immutability."

## Output Format

```yaml
root_cause: "push_failure â€” <specific_cause>"
evidence:
  - type: auth_status
    content: "<login result>"
  - type: repository_config
    content: "<describe-repositories output>"
severity: HIGH
mitigation:
  immediate: "Fix authentication or permissions"
  long_term: "Automate token refresh in CI/CD pipeline"
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
