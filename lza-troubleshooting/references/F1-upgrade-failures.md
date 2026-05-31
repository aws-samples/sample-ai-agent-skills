---
title: "F1 — Upgrade Failures"
description: "Diagnose and resolve LZA version upgrade failures"
status: active
severity: CRITICAL
triggers: ["LZA upgrade failed", "version upgrade error", "accelerator upgrade issue", "upgrade pipeline failed"]
owner: devops-agent
objective: "Successfully upgrade LZA to a newer version while maintaining operational stability"
context: "LZA upgrades involve updating the installer stack with the new version, which triggers a pipeline run that updates all CloudFormation stacks. Upgrades can fail due to version incompatibility, breaking changes, or state migration issues. Sequential version upgrades are required."
---

## Phase 1 — Triage

MUST:
- Check current LZA version: `aws cloudformation describe-stacks --stack-name AWSAccelerator-InstallerStack --query 'Stacks[0].Parameters[?ParameterKey==`AcceleratorVersion`].ParameterValue'`
- Check pipeline status after upgrade: `aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline`
- Review CodeBuild logs for upgrade-related errors

SHOULD:
- Check the target version's release notes for breaking changes
- Verify configuration file compatibility with the new version
- Check for deprecated configuration properties

MAY:
- Compare CloudFormation template changes between versions
- Review the LZA GitHub repository for known upgrade issues

## Phase 2 — Remediate

MUST:
- Follow the documented sequential upgrade path (no version skipping)
- Update configuration files for any schema changes in the new version
- Fix breaking changes identified in the release notes

SHOULD:
- Back up all configuration files before upgrading
- Test the upgrade in a non-production environment first
- Review the CDK diff output before approving deployment

MAY:
- Update configuration files incrementally for each version step
- Check for new required configuration properties in the target version

## Safety Ratings

```
safety_ratings:
  - "cloudformation describe-stacks: GREEN — read-only stack check"
  - "codepipeline get-pipeline-state: GREEN — read-only pipeline status"
  - "LZA version upgrade: RED — modifies infrastructure across all accounts"
  - "Configuration file backup: GREEN — read-only file copy"
```

## Escalation Conditions

- Upgrade fails with state migration errors that cannot be resolved
- Breaking changes require significant configuration rewrites
- Upgrade partially applied leaving stacks in inconsistent states
- Rollback to previous version is not straightforward

## Data Sensitivity

- Current LZA version: LOW — version information only
- Configuration files: HIGH — contain organization structure and security settings
- Upgrade release notes: LOW — publicly available information
- CloudFormation template diffs: MEDIUM — reveal infrastructure changes

## Prohibited Actions

- NEVER skip LZA versions during upgrade
- NEVER upgrade LZA in production without testing in a non-production environment
- NEVER approve the pipeline without reviewing the CDK diff
- NEVER upgrade during business-critical periods

## Phase 3 — Rollback

1. If upgrade failed, update the installer stack back to the previous version
2. Restore configuration files from backup if they were modified for the upgrade
3. Re-run the pipeline with the previous version to restore infrastructure
4. Verify all stacks are in a healthy state after rollback
5. Document the upgrade failure and root cause for future attempts

## Common Issues

- symptoms: "Pipeline fails after upgrade with 'Schema validation error'"
  diagnosis: "Configuration file schema changed in the new version. Properties were added, removed, or renamed."
  resolution: "Review release notes for schema changes. Update configuration files to match the new schema."

- symptoms: "Upgrade succeeds but stacks fail with resource errors"
  diagnosis: "New version creates or modifies resources that conflict with existing infrastructure."
  resolution: "Review stack events for specific resource errors. Fix conflicts in configuration files."

## Output Format

```yaml
root_cause: "upgrade — <specific_cause>"
evidence:
  - type: upgrade_error
    content: "<version details, pipeline status, and error information>"
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
