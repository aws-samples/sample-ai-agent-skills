---
title: "C2 — Security Config Errors"
description: "Diagnose and resolve security-config.yaml configuration and deployment errors"
status: active
severity: HIGH
triggers: ["security config error", "GuardDuty error", "Security Hub error", "security-config.yaml invalid", "Config rule error"]
owner: devops-agent
objective: "Fix security-config.yaml errors to enable security service deployment across accounts"
context: "The security-config.yaml controls GuardDuty, Security Hub, Macie, AWS Config rules, CloudTrail, access analyzer, and other security services. Errors affect security monitoring across all accounts. Security configuration changes have organization-wide impact."
---

## Phase 1 — Triage

MUST:
- Download and review security-config.yaml: `aws s3 cp s3://aws-accelerator-config-<account-id>-<region>/security-config.yaml ./`
- Check CloudFormation stack events for security stacks: `aws cloudformation describe-stack-events --stack-name AWSAccelerator-SecurityStack-<account>-<region> --max-items 20`
- Verify security service availability in target regions

SHOULD:
- Check GuardDuty status: `aws guardduty list-detectors`
- Check Security Hub status: `aws securityhub describe-hub`
- Review CodeBuild logs for the Security stage

MAY:
- Check Config rule compliance: `aws configservice describe-compliance-by-config-rule`
- Verify CloudTrail configuration: `aws cloudtrail describe-trails`

## Phase 2 — Remediate

MUST:
- Fix YAML syntax and schema errors in security-config.yaml
- Ensure security services are available in the target regions
- Verify IAM permissions for security service deployment

SHOULD:
- Check for conflicts with existing security service configurations
- Verify delegated administrator accounts are correctly configured
- Ensure Config rule definitions are valid

MAY:
- Test security configuration in a single account before organization-wide deployment
- Review security service quotas in target accounts

## Safety Ratings

```
safety_ratings:
  - "guardduty list-detectors: GREEN — read-only detector listing"
  - "securityhub describe-hub: GREEN — read-only hub status"
  - "configservice describe-compliance: GREEN — read-only compliance check"
  - "Security config deployment: RED — enables/modifies security services across all accounts"
```

## Escalation Conditions

- Security service deployment would conflict with existing third-party security tools
- Config rule changes affect compliance reporting for regulated workloads
- GuardDuty or Security Hub changes require security team approval
- Delegated administrator changes affect organization-level security management

## Data Sensitivity

- security-config.yaml: HIGH — reveals complete security architecture
- GuardDuty findings: HIGH — contain security threat information
- Security Hub findings: HIGH — contain compliance and security data
- Config rule compliance data: MEDIUM — reveals resource compliance status

## Prohibited Actions

- NEVER disable security services (GuardDuty, Security Hub) in production without approval
- NEVER remove Config rules that enforce compliance requirements
- NEVER modify CloudTrail configuration to reduce logging scope
- NEVER deploy security changes without security team review

## Phase 3 — Rollback

1. If security services were incorrectly disabled, re-enable them immediately
2. If Config rules were removed, restore them from the previous configuration
3. If delegated administrator was changed, revert to the previous admin account
4. Document all security configuration changes and their impact
5. Notify the security team about any changes to security monitoring

## Common Issues

- symptoms: "GuardDuty deployment fails with 'Detector already exists'"
  diagnosis: "GuardDuty was previously enabled manually or by another tool in the target account."
  resolution: "Import existing detector or disable the manual configuration before LZA deployment."

- symptoms: "Config rule creation fails with 'MaxNumberOfConfigRulesExceededException'"
  diagnosis: "AWS Config rule limit reached in the target account."
  resolution: "Request Config rule quota increase or consolidate rules."

## Output Format

```yaml
root_cause: "security_config — <specific_cause>"
evidence:
  - type: security_error
    content: "<security service status, stack events, and configuration>"
severity: HIGH
mitigation:
  immediate: "Fix security-config.yaml"
  long_term: "Implement security configuration validation and testing"
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
