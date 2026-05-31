---
title: "A2 — ECR Scanning"
description: "Diagnose ECR image scanning issues with Inspector"
status: active
severity: HIGH
triggers:
  - "ECR not scanned"
  - "image scanning failed"
  - "ECR findings missing"
  - "enhanced scanning not working"
owner: devops-agent
objective: "Ensure ECR images are being scanned by Inspector enhanced scanning"
context: "Inspector enhanced scanning for ECR replaces basic ECR scanning. It provides continuous monitoring and rescanning when new CVEs are published. Issues arise when enhanced scanning is not enabled, repositories use basic scanning, or images are in unsupported formats."
---

## Phase 1 — Triage

MUST:
- Check Inspector ECR scanning status: `aws inspector2 batch-get-account-status --query 'Accounts[*].{Account:AccountId,ECR:ResourceState.Ecr.Status}'`
- Check ECR repository scan configuration: `aws ecr describe-repositories --repository-names <repo-name> --query 'repositories[*].{Name:repositoryName,ScanConfig:imageScanningConfiguration}'`
- Check ECR scan type (basic vs enhanced): `aws ecr get-registry-scanning-configuration`
- List ECR coverage: `aws inspector2 list-coverage --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_CONTAINER_IMAGE"}]}'`

SHOULD:
- Check if specific images have scan findings: `aws inspector2 list-findings --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_CONTAINER_IMAGE"}]}' --max-results 10`
- Verify the image was pushed after enhanced scanning was enabled
- Check for scan errors on specific images

MAY:
- Check if the image format is supported (Docker V2, OCI)
- Verify ECR repository permissions

## Phase 2 — Remediate

MUST:
- Enable enhanced scanning: `aws ecr put-registry-scanning-configuration --scan-type ENHANCED --rules '[{"repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}],"scanFrequency":"SCAN_ON_PUSH"}]'`
- Enable Inspector ECR scanning: `aws inspector2 enable --resource-types ECR`

SHOULD:
- Push a new image to trigger initial scan after enabling
- Verify findings appear after scanning completes
- Configure scan-on-push for all repositories

MAY:
- Set up scan filters for specific repository patterns
- Configure continuous scanning frequency

## Common Issues

- symptoms: "ECR images show no Inspector findings"
  diagnosis: "Enhanced scanning not enabled — still using basic scanning."
  resolution: "Switch to enhanced scanning in ECR registry settings."

- symptoms: "Some repositories scanned but not others"
  diagnosis: "Scanning rules only match specific repository patterns."
  resolution: "Update scanning rules to include all target repositories."

- symptoms: "Image scanned but findings seem incomplete"
  diagnosis: "Image uses unsupported OS or package manager."
  resolution: "Check supported OS list. Use supported base images."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable enhanced scanning | YELLOW | Changes ECR scanning mode for all repositories |
| Enable Inspector ECR scanning | GREEN | Enables vulnerability detection |
| Push new image to trigger scan | GREEN | Normal ECR operation |
| Update scanning rules | YELLOW | Affects which repositories are scanned |

## Escalation Conditions

- Production ECR images not being scanned — escalate to security operations
- Switching from basic to enhanced scanning — escalate to ECR repository owners
- Scanning configuration changes affecting organization — escalate to delegated admin
- Critical vulnerabilities found in production images — escalate to application team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status |
| `aws ecr get-registry-scanning-configuration` | MEDIUM | Exposes scanning mode and rules |
| `aws inspector2 list-coverage` (ECR) | MEDIUM | Exposes image coverage status |
| `aws inspector2 list-findings` (ECR) | MEDIUM | Exposes vulnerability findings |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** revert from enhanced to basic scanning without security team approval
- **NEVER** ignore critical CVEs in production container images

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled enhanced scanning | Revert to basic scanning: `aws ecr put-registry-scanning-configuration --scan-type BASIC` |
| Enabled Inspector ECR scanning | Disable: `aws inspector2 disable --resource-types ECR` |
| Updated scanning rules | Restore previous scanning rules |

## Output Format

```yaml
root_cause: "ecr_scanning — <specific_cause>"
evidence:
  - type: scanning_config
    content: "<ECR scanning configuration>"
  - type: coverage
    content: "<ECR image coverage>"
severity: HIGH
mitigation:
  immediate: "Enable enhanced scanning and verify coverage"
  long_term: "Automate scanning configuration for all repositories"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
