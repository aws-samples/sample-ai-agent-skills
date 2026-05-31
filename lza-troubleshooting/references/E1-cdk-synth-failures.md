---
title: "E1 — CDK Synth Failures"
description: "Diagnose and resolve CDK synthesis failures in the LZA pipeline"
status: active
severity: HIGH
triggers: ["CDK synth failed", "synthesis error", "cdk synth error", "template generation failed"]
owner: devops-agent
objective: "Resolve CDK synthesis errors to enable CloudFormation template generation"
context: "CDK synthesis is the first deployment phase — it parses LZA configuration files and generates CloudFormation templates. Synth failures indicate configuration parsing errors, schema violations, or CDK code issues. No infrastructure changes occur during synth."
---

## Phase 1 — Triage

MUST:
- Check the Build stage of the pipeline: `aws codepipeline get-pipeline-state --name AWSAccelerator-Pipeline --query 'stageStates[?stageName==`Build`]'`
- Get CodeBuild build details: `aws codebuild batch-get-builds --ids <build-id>`
- Review CodeBuild logs for CDK synth error messages

SHOULD:
- Identify which configuration file caused the synth failure
- Check for YAML syntax errors in all configuration files
- Verify configuration file schema matches the LZA version

MAY:
- Run CDK synth locally to reproduce the error
- Compare configuration files with the LZA schema documentation

## Phase 2 — Remediate

MUST:
- Fix configuration file errors identified in the CDK synth output
- Correct YAML syntax issues (indentation, quoting, special characters)
- Ensure all referenced resources and accounts exist

SHOULD:
- Validate all configuration files against the LZA schema
- Check for cross-file reference consistency (e.g., OU names in accounts-config match organization-config)
- Verify the LZA version matches the configuration file schema version

MAY:
- Use a YAML linter for syntax validation
- Implement pre-commit hooks for configuration validation

## Safety Ratings

```
safety_ratings:
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "codebuild batch-get-builds: GREEN — read-only build details"
  - "CDK synth: GREEN — generates templates only, no infrastructure changes"
  - "Configuration file updates: YELLOW — changes deployed on next pipeline run"
```

## Escalation Conditions

- CDK synth fails with internal CDK errors (not configuration errors)
- Configuration schema has changed between LZA versions requiring migration
- Synth failure cannot be reproduced locally
- Multiple configuration files have interdependent errors

## Data Sensitivity

- CodeBuild logs: MEDIUM — contain configuration parsing output
- CDK synth output: MEDIUM — reveals generated CloudFormation template structure
- Configuration files: HIGH — contain organization structure and security settings
- CDK error messages: LOW — technical error details

## Prohibited Actions

- NEVER modify CDK code directly to fix configuration issues
- NEVER skip CDK synth validation by forcing deployment
- NEVER ignore synth warnings — they may indicate future failures
- NEVER commit configuration changes without validating YAML syntax

## Phase 3 — Rollback

1. If configuration changes caused synth failures, revert to the previous working configuration
2. If LZA version upgrade caused synth failures, check migration guide for schema changes
3. If synth passes but generates incorrect templates, review the CDK diff before approving
4. Document the synth failure and configuration fix
5. Verify synth succeeds before proceeding with deployment

## Common Issues

- symptoms: "CDK synth fails with 'Validation error: Invalid property'"
  diagnosis: "Configuration file uses a property not supported by the current LZA version."
  resolution: "Check LZA documentation for supported properties. Remove or rename invalid properties."

- symptoms: "CDK synth fails with 'Reference error: Account not found'"
  diagnosis: "A configuration file references an account name not defined in accounts-config.yaml."
  resolution: "Verify account names are consistent across all configuration files."

## Output Format

```yaml
root_cause: "cdk_synth — <specific_cause>"
evidence:
  - type: synth_error
    content: "<CDK error output and configuration file details>"
severity: HIGH
mitigation:
  immediate: "Fix configuration file errors"
  long_term: "Implement configuration validation pipeline"
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
