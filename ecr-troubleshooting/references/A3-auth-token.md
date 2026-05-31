---
title: "A3 â€” Auth Token Issues"
description: "Diagnose ECR authentication token problems including expiry, credential helpers, and cross-account auth"
status: active
severity: HIGH
triggers:
  - "ecr auth token"
  - "ecr login failed"
  - "authorization token expired"
  - "no basic auth credentials"
  - "ecr credential helper"
owner: devops-agent
objective: "Resolve ECR authentication token issues and establish reliable auth workflow"
context: "ECR uses temporary authorization tokens valid for 12 hours. Common issues include token expiry in long-running CI/CD jobs, credential helper misconfiguration, cross-account auth failures, and Docker config.json corruption."
---

## Phase 1 â€” Triage

MUST:
- Test current auth status: `aws ecr get-authorization-token --query 'authorizationData[0].expiresAt'`
- Verify AWS credentials are valid: `aws sts get-caller-identity`
- Check Docker config for stored credentials: look at `~/.docker/config.json`
- Verify IAM permissions include `ecr:GetAuthorizationToken`
- Check if credential helper is configured: look for `credHelpers` or `credsStore` in Docker config

SHOULD:
- Verify the correct region is being used for auth
- Check if STS endpoint is accessible (for assumed roles)
- Verify no proxy or firewall blocking ECR endpoints

MAY:
- Check if Amazon ECR Credential Helper is installed: `docker-credential-ecr-login version`
- Review CI/CD pipeline logs for auth timing issues

## Phase 2 â€” Remediate

MUST:
- Re-authenticate: `aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com`
- If using credential helper, verify config in `~/.docker/config.json`: `{"credHelpers":{"<account>.dkr.ecr.<region>.amazonaws.com":"ecr-login"}}`
- If cross-account, ensure role assumption works before ECR auth

SHOULD:
- Install Amazon ECR Credential Helper for automatic token refresh
- Add token refresh step at the start of CI/CD jobs
- Use instance profiles or IRSA for EKS-based workloads

## Common Issues

- symptoms: "no basic auth credentials"
  diagnosis: "Docker has no stored credentials for the ECR registry."
  resolution: "Run get-login-password and docker login."

- symptoms: "token expired during long build"
  diagnosis: "Build took longer than 12 hours, token expired mid-job."
  resolution: "Refresh token before push step, not just at job start."

- symptoms: "credential helper not found"
  diagnosis: "docker-credential-ecr-login not installed or not in PATH."
  resolution: "Install ECR credential helper and ensure it is in system PATH."

## Output Format

```yaml
root_cause: "auth_token_issue â€” <specific_cause>"
evidence:
  - type: token_status
    content: "<authorization token expiry>"
  - type: caller_identity
    content: "<sts get-caller-identity output>"
severity: HIGH
mitigation:
  immediate: "Re-authenticate to ECR"
  long_term: "Use ECR credential helper for automatic token management"
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
