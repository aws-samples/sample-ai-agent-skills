---
title: "D1 — Global Config Issues"
description: "Diagnose and resolve global-config.yaml configuration errors"
status: active
severity: HIGH
triggers: ["global config error", "global-config.yaml invalid", "logging config error", "home region error"]
owner: devops-agent
objective: "Fix global-config.yaml errors to enable correct global settings and logging configuration"
context: "The global-config.yaml defines global settings including home region, enabled regions, centralized logging, CloudTrail organization trail, S3 lifecycle policies, budget alerts, and SNS topics. Errors affect the foundational settings for the entire LZA deployment."
---

## Phase 1 — Triage

MUST:
- Download and review global-config.yaml: `aws s3 cp s3://aws-accelerator-config-<account-id>-<region>/global-config.yaml ./`
- Verify home region setting matches the LZA installation region
- Check enabled regions list for validity

SHOULD:
- Verify centralized logging bucket configuration
- Check CloudTrail organization trail settings
- Review CodeBuild logs for global configuration stage errors

MAY:
- Check S3 lifecycle policies for logging buckets
- Verify SNS topic configuration for budget alerts
- Review budget alert thresholds

## Phase 2 — Remediate

MUST:
- Fix YAML syntax and schema errors in global-config.yaml
- Ensure home region matches the LZA installation region
- Verify all enabled regions are valid AWS region codes

SHOULD:
- Validate centralized logging S3 bucket configuration
- Ensure CloudTrail organization trail settings are correct
- Check SNS topic ARNs for budget notifications

MAY:
- Review and optimize S3 lifecycle policies for cost management
- Implement budget alerts for cost monitoring

## Safety Ratings

```
safety_ratings:
  - "s3 cp (config download): GREEN — read-only file download"
  - "Global config deployment: YELLOW — modifies global settings across the organization"
  - "CloudTrail organization trail: YELLOW — affects logging for all accounts"
  - "S3 lifecycle policies: YELLOW — affects log retention"
```

## Escalation Conditions

- Home region change requires complete LZA reinstallation
- Centralized logging configuration affects compliance requirements
- CloudTrail changes affect audit trail for regulated workloads
- Budget alert changes require finance team approval

## Data Sensitivity

- global-config.yaml: MEDIUM — contains global settings and logging configuration
- CloudTrail configuration: MEDIUM — reveals logging scope and destinations
- S3 bucket configuration: MEDIUM — reveals logging architecture
- Budget configuration: LOW — cost thresholds and notification settings

## Prohibited Actions

- NEVER change the home region after initial installation
- NEVER disable centralized logging without compliance team approval
- NEVER reduce CloudTrail logging scope in production
- NEVER delete logging S3 buckets

## Phase 3 — Rollback

1. If global settings were changed incorrectly, revert global-config.yaml and re-run the pipeline
2. If logging configuration was modified, verify logs are still being collected
3. If CloudTrail settings were changed, verify trail is active in all accounts
4. Document all global configuration changes
5. Verify logging and monitoring are operational after changes

## Common Issues

- symptoms: "Pipeline fails with 'Invalid region' error"
  diagnosis: "An enabled region in global-config.yaml is not a valid AWS region code."
  resolution: "Check region codes. Use standard format (e.g., us-east-1, eu-west-1). Remove invalid regions."

- symptoms: "Centralized logging bucket creation fails"
  diagnosis: "S3 bucket name conflict or KMS key configuration error."
  resolution: "Check S3 bucket naming. Verify KMS key exists and has correct policy for cross-account access."

## Output Format

```yaml
root_cause: "global_config — <specific_cause>"
evidence:
  - type: config_error
    content: "<global settings, logging configuration, and error details>"
severity: HIGH
mitigation:
  immediate: "Fix global-config.yaml"
  long_term: "Implement global configuration validation"
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
