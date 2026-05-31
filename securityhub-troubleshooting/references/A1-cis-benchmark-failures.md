---
title: "A1 — CIS Benchmark Failures"
description: "Diagnose CIS AWS Foundations Benchmark control failures in Security Hub"
status: active
severity: HIGH
triggers:
  - "CIS benchmark"
  - "CIS control failed"
  - "CIS compliance"
  - "CIS foundations"
  - "CIS standard"
owner: devops-agent
objective: "Identify and resolve CIS AWS Foundations Benchmark control failures"
context: "The CIS AWS Foundations Benchmark is a set of security best practices for AWS accounts. Controls evaluate IAM, logging, monitoring, and networking configurations. Most controls require AWS Config to be enabled. Failures indicate security configuration gaps."
---

## Phase 1 — Triage

MUST:
- Check if CIS standard is enabled: `aws securityhub get-enabled-standards`
- List CIS controls and their status: `aws securityhub describe-standards-controls --standards-subscription-arn <cis-standard-arn>`
- Get failed CIS findings: `aws securityhub get-findings --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}],"ProductName":[{"Value":"Security Hub","Comparison":"EQUALS"}],"GeneratorId":[{"Value":"cis","Comparison":"PREFIX"}]}' --max-results 20`
- Verify Config is enabled: `aws configservice describe-configuration-recorder-status`
- Check which Config rules back the failing controls

SHOULD:
- Get specific control details for failing controls
- Verify Config records the resource types needed by CIS controls
- Check if CloudTrail is enabled (required for several CIS controls): `aws cloudtrail describe-trails`

MAY:
- Compare CIS v1.2 vs v1.4 vs v3.0 control differences
- Check for controls that require CloudWatch alarms or metric filters

## Phase 2 — Remediate

MUST:
- Enable Config recorder if not running
- Fix the specific resource configuration causing the control failure
- Verify the control passes after remediation

SHOULD:
- Enable CloudTrail with log file validation for CIS logging controls
- Set up CloudWatch metric filters and alarms for CIS monitoring controls
- Configure IAM password policy for CIS IAM controls

MAY:
- Create automated remediation for common CIS failures
- Set up compliance change notifications

## Common Issues

- symptoms: "Multiple CIS controls show NO_DATA status"
  diagnosis: "AWS Config is not enabled or not recording the required resource types."
  resolution: "Enable Config recorder with all supported resource types. Wait up to 24 hours for initial evaluation."

- symptoms: "CIS control for CloudTrail logging shows FAILED"
  diagnosis: "CloudTrail is not enabled, not logging to S3, or log file validation is disabled."
  resolution: "Enable CloudTrail in all regions with log file validation and S3 delivery."

- symptoms: "CIS IAM controls show FAILED for password policy"
  diagnosis: "The account password policy does not meet CIS requirements (length, complexity, rotation)."
  resolution: "Update the IAM password policy to meet CIS requirements: minimum 14 characters, require symbols, numbers, uppercase, lowercase."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Config recorder | YELLOW | Service enablement; reversible |
| Fix resource configuration | YELLOW | Resource change; varies by resource |
| Enable CloudTrail | YELLOW | Service enablement; reversible |
| Update IAM password policy | YELLOW | Policy change; reversible |

## Escalation Conditions

- Standard enablement/disablement
- Cross-account configuration changes
- Config recorder changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-enabled-standards` | LOW | Standard subscription status |
| `describe-standards-controls` | LOW | Control status |
| `get-findings` (CIS) | MEDIUM | Compliance findings with resource details |
| `describe-configuration-recorder-status` | LOW | Recorder status |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings
- NEVER disable CIS controls without documented justification

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Config recorder enablement | Stop recorder (not recommended) |
| Resource configuration fix | Revert resource to previous configuration |
| CloudTrail enablement | Disable trail (not recommended) |
| IAM password policy change | Revert policy to previous values |

## Output Format

```yaml
root_cause: "cis_benchmark_failure — <specific_control>"
evidence:
  - type: standard_controls
    content: "<CIS control status>"
  - type: findings
    content: "<failed CIS findings>"
  - type: config_status
    content: "<Config recorder status>"
severity: HIGH
mitigation:
  immediate: "Fix the specific control failure"
  long_term: "Implement automated remediation and continuous monitoring"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
