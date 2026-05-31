## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix artifact S3 bucket permissions | YELLOW |
| Verify artifact encryption configuration | GREEN |
| Check artifact size limits | GREEN |
| Fix artifact path configuration in pipeline | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Artifacts may contain compiled application code, deployment packages, and configuration files

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If S3 bucket permissions were modified, revert to the previous bucket policy
2. If artifact encryption was changed, restore the previous KMS key configuration
3. If artifact path configuration was changed in the pipeline, restore the previous pipeline definition
4. Verify rollback by triggering a pipeline execution and confirming artifacts upload/download successfully

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
