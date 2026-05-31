---
title: "B1 â€” Image Scanning Issues"
description: "Diagnose ECR image scanning failures including basic vs enhanced scanning and scan findings"
status: active
severity: MEDIUM
triggers:
  - "image scan failed"
  - "scan not completing"
  - "vulnerability scan"
  - "ecr scanning"
  - "scan findings"
owner: devops-agent
objective: "Resolve image scanning issues and ensure vulnerability data is available"
context: "ECR supports basic scanning (Clair-based, on-push) and enhanced scanning (Inspector-based, continuous). Common issues include scanning not enabled, scan failures on specific images, and misunderstanding scan modes."
---

## Phase 1 â€” Triage

MUST:
- Check scanning configuration: `aws ecr get-registry-scanning-configuration`
- Check repository scan-on-push setting: `aws ecr describe-repositories --repository-names <repo> --query 'repositories[0].imageScanningConfiguration'`
- Get scan findings: `aws ecr describe-image-scan-findings --repository-name <repo> --image-id imageTag=<tag>`
- Check scan status: `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag> --query 'imageDetails[0].imageScanStatus'`

SHOULD:
- Verify if enhanced scanning is enabled (requires Inspector): `aws inspector2 list-coverage --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_CONTAINER_IMAGE"}]}'`
- Check for unsupported image formats (scanning requires Linux AMD64/ARM64 images)

MAY:
- Start a manual scan: `aws ecr start-image-scan --repository-name <repo> --image-id imageTag=<tag>`
- Check Inspector findings if enhanced scanning: `aws inspector2 list-findings --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_CONTAINER_IMAGE"}]}'`

## Phase 2 â€” Remediate

MUST:
- Enable scan-on-push if disabled: `aws ecr put-image-scanning-configuration --repository-name <repo> --image-scanning-configuration scanOnPush=true`
- For enhanced scanning, enable Inspector ECR scanning: `aws inspector2 enable --resource-types ECR`
- If scan failed, check image format compatibility

SHOULD:
- Configure scan filters for enhanced scanning to control which repositories are scanned
- Set up EventBridge rules for scan completion notifications

## Common Issues

- symptoms: "scan status SCAN_ELIGIBILITY_EXPIRED"
  diagnosis: "Basic scanning only allows one scan per image per 24 hours."
  resolution: "Wait 24 hours or switch to enhanced scanning for continuous monitoring."

- symptoms: "UNSUPPORTED_IMAGE"
  diagnosis: "Image OS or architecture not supported by scanner."
  resolution: "Ensure images are Linux-based AMD64 or ARM64."

## Output Format

```yaml
root_cause: "scanning_issue â€” <specific_cause>"
evidence:
  - type: scan_config
    content: "<scanning configuration>"
  - type: scan_status
    content: "<image scan status>"
severity: MEDIUM
mitigation:
  immediate: "Enable scanning or fix image compatibility"
  long_term: "Enable enhanced scanning for continuous vulnerability monitoring"
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
