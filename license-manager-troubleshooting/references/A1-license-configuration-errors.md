---
title: "A1 — License Configuration Errors"
description: "Diagnose license configuration creation and management errors"
status: active
severity: HIGH
triggers: ["license configuration error", "cannot create license config", "license config failed"]
owner: devops-agent
objective: "Resolve license configuration creation and management issues"
context: "License configurations define counting rules and limits for software licenses. They track consumption against defined limits."
---

## Phase 1 — Triage

MUST:
- List existing configurations: `aws license-manager list-license-configurations`
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names license-manager:CreateLicenseConfiguration`
- Verify License Manager service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForLicenseManager`

SHOULD:
- Check CloudTrail for configuration errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=license-manager.amazonaws.com --max-results 20`
- Verify the counting type matches the software license agreement

MAY:
- Review existing configurations for conflicts
- Check if the configuration name already exists

## Phase 2 — Remediate

MUST:
- Create service-linked role if missing: `aws license-manager create-service-linked-role`
- Fix IAM permissions for license-manager:CreateLicenseConfiguration
- Use correct counting type (vCPU, Instance, Core, Socket) per vendor agreement

SHOULD:
- Set appropriate license count and hard/soft limit: `aws license-manager create-license-configuration --name <name> --license-counting-type vCPU --license-count 100 --license-count-hard-limit`
- Associate AMIs with the configuration for automatic tracking

MAY:
- Set up CloudWatch alarms for license consumption thresholds
- Tag configurations for cost allocation

## Safety Ratings

```
safety_ratings:
  - "license-manager list-license-configurations: GREEN — read-only listing"
  - "license-manager create-license-configuration: YELLOW — creates tracking resource"
  - "license-manager create-service-linked-role: YELLOW — creates IAM role"
  - "iam simulate-principal-policy: GREEN — read-only permission check"
```

## Escalation Conditions

- Service-linked role creation fails with permission errors
- License configuration creation fails with internal service errors
- Counting type unclear from vendor license agreement
- Configuration needed for compliance audit deadline

## Data Sensitivity

- `aws license-manager list-license-configurations`: LOW — returns configuration metadata
- License counts: MEDIUM — reveals software licensing posture
- Vendor license agreements: HIGH — contains contractual terms
- `aws cloudtrail lookup-events`: MEDIUM — reveals user actions

## Prohibited Actions

- NEVER delete license configurations to hide compliance violations
- NEVER set license counts higher than actual purchased licenses
- NEVER disable hard limits without compliance team approval
- NEVER create configurations with wrong counting types to avoid violations

## Phase 3 — Rollback

1. Delete incorrectly created configurations: `aws license-manager delete-license-configuration --license-configuration-arn <arn>`
2. Remove incorrect AMI associations: `aws license-manager update-license-configuration --license-configuration-arn <arn> --remove-license-specifications`
3. If service-linked role was created incorrectly, it can be deleted via IAM (only if no configurations exist)
4. Verify no resources are left untracked after configuration changes
5. Document configuration changes for compliance records

## Common Issues

- symptoms: "CreateLicenseConfiguration returns ServiceLinkedRoleNotFound"
  diagnosis: "License Manager service-linked role has not been created."
  resolution: "Run aws license-manager create-service-linked-role or create via IAM console."

## Output Format

```yaml
root_cause: "license_configuration — <specific_cause>"
evidence:
  - type: configuration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix configuration issue"
  long_term: "Automate license configuration via IaC"
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
