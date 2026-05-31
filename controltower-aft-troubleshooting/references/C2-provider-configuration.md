---
title: "C2 — Provider Configuration"
description: "Diagnose and resolve Terraform provider configuration issues in AFT"
status: active
severity: HIGH
triggers: ["provider error", "provider version mismatch", "AWS provider configuration", "provider initialization failed"]
owner: devops-agent
objective: "Ensure Terraform providers are correctly configured and compatible with AFT"
context: "AFT uses the AWS Terraform provider with specific version constraints. Provider issues include version mismatches, authentication failures, and region configuration errors. The provider must be configured to assume roles in target accounts."
---

## Phase 1 — Triage

MUST:
- Check Terraform and provider versions: `terraform version`
- Review `versions.tf` for provider version constraints
- Check `.terraform.lock.hcl` for locked provider versions

SHOULD:
- Verify provider authentication: `aws sts get-caller-identity`
- Check if provider can assume the target account role
- Review CodeBuild logs for provider initialization errors

MAY:
- Compare provider versions across different AFT repositories
- Check Terraform registry for provider version compatibility

## Phase 2 — Remediate

MUST:
- Fix provider version constraints in `versions.tf` to match AFT requirements
- Run `terraform init -upgrade` to update provider plugins
- Ensure provider configuration includes correct assume_role blocks for target accounts

SHOULD:
- Update `.terraform.lock.hcl` after provider version changes
- Verify provider region configuration matches the deployment region
- Check for deprecated provider features in the current AFT Terraform code

MAY:
- Pin provider versions to specific minor versions for stability
- Set up a Terraform provider mirror for air-gapped environments

## Safety Ratings

```
safety_ratings:
  - "terraform version: GREEN — read-only version check"
  - "terraform init -upgrade: YELLOW — downloads and updates provider plugins"
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "Provider version changes: YELLOW — may affect Terraform behavior"
```

## Escalation Conditions

- Provider version required by AFT is not available in the Terraform registry
- Provider authentication fails due to STS assume-role chain issues
- Provider version upgrade introduces breaking changes in AFT Terraform code
- Air-gapped environment cannot download provider plugins

## Data Sensitivity

- `terraform version`: LOW — version information only
- Provider configuration: MEDIUM — contains role ARNs and region settings
- `.terraform.lock.hcl`: LOW — provider hashes and versions
- `aws sts get-caller-identity`: LOW — caller identity only

## Prohibited Actions

- NEVER remove provider version constraints to "fix" version issues
- NEVER use provider versions newer than what AFT has been tested with
- NEVER hardcode AWS credentials in provider configuration
- NEVER skip `terraform init` after changing provider versions

## Phase 3 — Rollback

1. If provider upgrade caused issues, revert `versions.tf` and `.terraform.lock.hcl` to previous versions
2. Run `terraform init` to restore previous provider versions
3. If provider configuration changes broke assume-role, revert the provider block
4. Document provider version changes and their impact
5. Verify all AFT pipelines work with the reverted provider versions

## Common Issues

- symptoms: "Error: Failed to query available provider packages"
  diagnosis: "Network connectivity issue preventing Terraform from reaching the registry."
  resolution: "Check internet access from CodeBuild. If in VPC, verify NAT gateway. Consider provider mirror."

- symptoms: "Error: Incompatible provider version"
  diagnosis: "Provider version constraint in versions.tf conflicts with the lock file or AFT requirements."
  resolution: "Update version constraints. Run `terraform init -upgrade`. Verify compatibility with AFT module."

## Output Format

```yaml
root_cause: "provider_configuration — <specific_cause>"
evidence:
  - type: provider_error
    content: "<provider version, configuration, and error details>"
severity: HIGH
mitigation:
  immediate: "Fix provider version or configuration"
  long_term: "Pin provider versions and implement version compatibility testing"
```

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
