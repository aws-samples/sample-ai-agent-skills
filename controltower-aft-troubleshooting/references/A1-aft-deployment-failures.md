---
title: "A1 — AFT Deployment Failures"
description: "Diagnose and resolve AFT initial deployment and Terraform apply failures"
status: active
severity: CRITICAL
triggers: ["AFT deployment failed", "terraform apply error", "AFT installation failed", "AFT bootstrap error"]
owner: devops-agent
objective: "Successfully deploy or redeploy AFT infrastructure in the AFT management account"
context: "AFT deployment uses Terraform to create infrastructure in the AFT management account including CodePipeline pipelines, Lambda functions, DynamoDB tables, S3 buckets, and IAM roles. Deployment failures are typically caused by IAM permissions, Terraform version mismatches, or missing prerequisites."
---

## Phase 1 — Triage

MUST:
- Check Terraform version compatibility: `terraform version`
- Review Terraform plan/apply output for specific error messages
- Verify the AFT management account is enrolled in Control Tower: `aws organizations describe-account --account-id <aft-account-id>`

SHOULD:
- Check IAM permissions for the deployment role: `aws sts get-caller-identity`
- Verify Control Tower is set up and landing zone is active: `aws controltower list-landing-zones`
- Check for existing AFT resources that may conflict: `aws codepipeline list-pipelines --query 'pipelines[?contains(name,`aft`)]'`

MAY:
- Review the AFT module version in `main.tf` for compatibility
- Check AWS provider version constraints in `versions.tf`
- Verify S3 backend bucket exists for Terraform state

## Phase 2 — Remediate

MUST:
- Fix Terraform version to match AFT requirements (check AFT module documentation)
- Ensure the deployment IAM role has AdministratorAccess or equivalent in the AFT management account
- Resolve any Terraform provider version conflicts

SHOULD:
- Run `terraform init -upgrade` to refresh provider plugins
- Check and fix the AFT module source reference and version pin
- Verify all required AFT input variables are set correctly

MAY:
- Use `terraform plan -target` to deploy specific resources that failed
- Enable Terraform debug logging: `TF_LOG=DEBUG terraform apply`

## Safety Ratings

```
safety_ratings:
  - "terraform plan: GREEN — read-only preview of changes"
  - "terraform apply: RED — creates/modifies infrastructure resources"
  - "organizations describe-account: GREEN — read-only account check"
  - "codepipeline list-pipelines: GREEN — read-only pipeline listing"
```

## Escalation Conditions

- Deployment fails with Control Tower API errors requiring CT admin intervention
- IAM permissions cannot be granted due to SCP restrictions
- Terraform state is corrupted from a partial deployment
- AFT module version is incompatible with the current Control Tower version

## Data Sensitivity

- Terraform state files: HIGH — contain resource ARNs, configuration, and potentially secrets
- AFT input variables: HIGH — may contain account emails, SSO configuration
- IAM role policies: MEDIUM — reveals permission scope
- `aws sts get-caller-identity`: LOW — returns caller identity only

## Prohibited Actions

- NEVER deploy AFT with AdministratorAccess in the Control Tower management account
- NEVER delete existing AFT infrastructure to "start fresh" without backing up state
- NEVER hardcode credentials in Terraform configuration files
- NEVER skip Terraform plan review before applying changes

## Phase 3 — Rollback

1. If partial deployment created resources, use `terraform destroy -target` for specific resources (with extreme caution)
2. If Terraform state is corrupted, restore from S3 versioning: `aws s3api list-object-versions --bucket <state-bucket> --prefix <state-key>`
3. If IAM roles were created with excessive permissions, scope them down after successful deployment
4. Document the deployment attempt and all configuration changes
5. Notify the Control Tower admin if organizational changes were made

## Common Issues

- symptoms: "Terraform apply fails with 'Error creating CodePipeline'"
  diagnosis: "IAM role for CodePipeline does not have sufficient permissions or the service-linked role is missing."
  resolution: "Ensure the deployment role has permissions to create CodePipeline resources and IAM roles."

- symptoms: "Module source error: 'Failed to download module'"
  diagnosis: "AFT module source URL is incorrect or network connectivity prevents download."
  resolution: "Verify the module source in main.tf. Ensure internet access from the deployment environment."

## Output Format

```yaml
root_cause: "aft_deployment — <specific_cause>"
evidence:
  - type: deployment_error
    content: "<Terraform error output>"
severity: CRITICAL
mitigation:
  immediate: "Fix deployment blocker"
  long_term: "Implement deployment pipeline with pre-flight checks"
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
