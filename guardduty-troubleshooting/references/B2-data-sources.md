---
title: "B2 — Data Source Issues"
description: "Diagnose GuardDuty data source configuration and ingestion issues"
status: active
severity: HIGH
triggers:
  - "data source"
  - "no findings generated"
  - "data source disabled"
  - "flow logs not analyzed"
  - "cloudtrail not analyzed"
owner: devops-agent
objective: "Identify and resolve data source configuration issues affecting GuardDuty finding generation"
context: "GuardDuty uses multiple data sources: VPC Flow Logs, DNS logs, CloudTrail management events, CloudTrail S3 data events, EKS audit logs, and runtime telemetry. Each data source generates specific finding types. Data source issues prevent corresponding findings from being generated."
---

## Phase 1 — Triage

MUST:
- Get detector configuration with data sources: `aws guardduty get-detector --detector-id <detector-id>`
- Check which data sources are enabled in the detector response
- Verify the detector status is ENABLED
- Check for findings from each data source type: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"service.additionalInfo.type":{"Eq":["<data-source>"]}}}' --max-results 5`

SHOULD:
- Verify the service-linked role has not been modified: `aws iam get-role --role-name AWSServiceRoleForAmazonGuardDuty`
- Check CloudWatch metrics for GuardDuty: `aws cloudwatch list-metrics --namespace AWS/GuardDuty`
- Review the detector's finding publishing frequency

MAY:
- Check if the account has VPC resources that would generate flow log findings
- Verify CloudTrail is logging (for correlation, not a GuardDuty prerequisite)

## Phase 2 — Remediate

MUST:
- Enable disabled data sources in the detector configuration
- Verify the service-linked role exists and has correct permissions
- Confirm findings start appearing after enabling data sources

SHOULD:
- Enable all data sources for comprehensive threat detection
- Set up monitoring for data source status changes

MAY:
- Create test scenarios to verify each data source generates findings
- Document which data sources are enabled and why

## Common Issues

- symptoms: "No network-based findings (Recon, Trojan, Backdoor)"
  diagnosis: "VPC Flow Log analysis may be disabled or the account has no VPC resources."
  resolution: "Verify the detector has VPC Flow Log analysis enabled. Check for VPC resources in the account."

- symptoms: "No S3-related findings"
  diagnosis: "S3 protection (CloudTrail S3 data events) is not enabled in the detector."
  resolution: "Enable S3 protection in the detector configuration."

- symptoms: "No EKS-related findings"
  diagnosis: "EKS Audit Log Monitoring is not enabled or there are no EKS clusters in the region."
  resolution: "Enable EKS Audit Log Monitoring in the detector. Verify EKS clusters exist."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable data sources | YELLOW | Configuration change; reversible |
| Verify service-linked role | GREEN | Read-only check |

## Escalation Conditions

- Detector configuration changes
- Data source enablement/disablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` | MEDIUM | Data source configuration |
| `list-findings` (by data source) | HIGH | Finding details |
| CloudWatch metrics | LOW | Aggregate metrics |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Data source enablement | Disable via `update-detector` (not recommended) |

## Output Format

```yaml
root_cause: "data_source_issue — <specific_cause>"
evidence:
  - type: detector_config
    content: "<detector data source configuration>"
  - type: findings
    content: "<finding types present/absent>"
  - type: service_role
    content: "<service-linked role status>"
severity: HIGH
mitigation:
  immediate: "Enable the missing data sources"
  long_term: "Monitor data source status and finding generation rates"
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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
