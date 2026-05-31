---
title: "G1 — AFT Upgrade Issues"
description: "Diagnose and resolve AFT version upgrade failures"
status: active
severity: CRITICAL
triggers: ["AFT upgrade failed", "version upgrade error", "module upgrade issue", "AFT version incompatible"]
owner: devops-agent
objective: "Successfully upgrade AFT to a newer version while maintaining operational stability"
context: "AFT upgrades require updating the Terraform module version, running terraform apply, and verifying compatibility with Control Tower and Terraform versions. Version skipping is not supported. Upgrades can fail due to version incompatibility, state migration issues, or breaking changes."
---

## Phase 1 — Triage

MUST:
- Check current AFT version in `main.tf` module source
- Review the target version's release notes and upgrade guide
- Verify Terraform version compatibility: `terraform version`

SHOULD:
- Check Control Tower version compatibility with the target AFT version
- Review AWS provider version requirements for the target AFT version
- Check for breaking changes in the upgrade path

MAY:
- Review the AFT GitHub repository for known upgrade issues
- Check if intermediate versions are required (no version skipping)

## Phase 2 — Remediate

MUST:
- Follow the documented upgrade path (no version skipping)
- Update module version in `main.tf` and run `terraform init -upgrade`
- Run `terraform plan` to review changes before applying
- Apply changes: `terraform apply` (after plan review)

SHOULD:
- Back up Terraform state before upgrading: `aws s3 cp s3://<state-bucket>/<state-key> ./state-backup.tfstate`
- Test the upgrade in a non-production AFT environment first
- Verify all AFT pipelines work after the upgrade

MAY:
- Update Terraform version if required by the new AFT version
- Update AWS provider version constraints if needed
- Review and update customization code for compatibility

## Safety Ratings

```
safety_ratings:
  - "terraform plan: GREEN — read-only preview of upgrade changes"
  - "terraform apply (upgrade): RED — modifies AFT infrastructure"
  - "s3 cp (state backup): GREEN — read-only state backup"
  - "terraform init -upgrade: YELLOW — updates provider and module plugins"
```

## Escalation Conditions

- Upgrade fails with state migration errors that cannot be resolved
- Breaking changes require significant customization code rewrites
- Control Tower version is incompatible and requires CT upgrade first
- Upgrade partially applied leaving AFT in an inconsistent state

## Data Sensitivity

- Terraform state backup: CRITICAL — contains all AFT resource configuration
- AFT module source and version: LOW — version information only
- Terraform plan output: HIGH — reveals infrastructure changes
- Upgrade documentation: LOW — publicly available information

## Prohibited Actions

- NEVER skip AFT versions during upgrade — follow the sequential upgrade path
- NEVER upgrade AFT without backing up Terraform state first
- NEVER upgrade AFT in production without testing in a non-production environment
- NEVER apply Terraform changes without reviewing the plan output

## Phase 3 — Rollback

1. If upgrade failed, restore Terraform state from backup: `aws s3 cp ./state-backup.tfstate s3://<state-bucket>/<state-key>`
2. Revert module version in `main.tf` to the previous version
3. Run `terraform init -upgrade` and `terraform apply` to restore previous version
4. Verify all AFT pipelines work after rollback
5. Document the upgrade failure and root cause for future attempts

## Common Issues

- symptoms: "Terraform apply fails with 'Error: Module version constraint not met'"
  diagnosis: "Terraform version is too old for the target AFT module version."
  resolution: "Upgrade Terraform to the version required by the target AFT module. Check AFT documentation."

- symptoms: "Upgrade succeeds but pipelines fail with new errors"
  diagnosis: "Breaking changes in the new AFT version require customization code updates."
  resolution: "Review release notes for breaking changes. Update customization code to match new API/schema."

## Output Format

```yaml
root_cause: "aft_upgrade — <specific_cause>"
evidence:
  - type: upgrade_error
    content: "<version details, Terraform error, and compatibility information>"
severity: CRITICAL
mitigation:
  immediate: "Fix upgrade blocker or rollback"
  long_term: "Implement upgrade testing pipeline and version compatibility checks"
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
