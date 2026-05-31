---
title: "D2 — Malware Scan Failures"
description: "Diagnose GuardDuty malware scan execution failures"
status: active
severity: HIGH
triggers:
  - "scan failed"
  - "malware scan error"
  - "scan not completing"
  - "snapshot creation failed"
  - "scan timeout"
owner: devops-agent
objective: "Identify and resolve malware scan execution failures"
context: "Malware scans can fail due to EBS snapshot creation errors, volume size limits, permission issues, encryption key problems, or scan timeouts. The scan status and failure reason provide diagnostic details."
---

## Phase 1 — Triage

MUST:
- List failed scans: `aws guardduty describe-malware-scans --detector-id <detector-id> --filter-criteria '{"FilterCriterion":[{"CriterionKey":"SCAN_STATUS","FilterCondition":{"EqualsValue":"FAILED"}}]}'`
- Check the scan failure reason in the scan details
- Verify the volume size is within scanning limits
- Check the EBS volume encryption: `aws ec2 describe-volumes --volume-ids <volume-id>`
- Verify the service-linked role permissions for EC2 and EBS

SHOULD:
- Check if the KMS key allows the GuardDuty service role: `aws kms describe-key --key-id <key-id>`
- Verify the EC2 instance is running and the volume is attached
- Check for scan exclusion tags on the volume or instance

MAY:
- Check EBS snapshot quota: `aws service-quotas get-service-quota --service-code ebs --quota-code L-309BACF6`
- Review the scan timeout settings

## Phase 2 — Remediate

MUST:
- Fix the identified failure cause (permissions, encryption, volume size)
- Ensure the KMS key policy allows the GuardDuty service-linked role
- Retry the scan or trigger a new on-demand scan

SHOULD:
- Update KMS key policies for customer-managed keys used on EBS volumes
- Configure scan exclusion tags for volumes that cannot be scanned

MAY:
- Request EBS snapshot quota increase if hitting limits
- Set up monitoring for scan failure rates

## Common Issues

- symptoms: "Scan failed with INTERNAL_ERROR"
  diagnosis: "An internal error occurred during snapshot creation or scan processing."
  resolution: "Retry the scan. If persistent, check volume state and permissions. Contact AWS Support if recurring."

- symptoms: "Scan failed due to volume size exceeding limit"
  diagnosis: "The EBS volume exceeds the maximum scannable size (default 1 TB)."
  resolution: "Volumes larger than the size limit cannot be scanned. Use alternative malware scanning tools for large volumes."

- symptoms: "Scan failed with KMS access denied"
  diagnosis: "The EBS volume is encrypted with a customer-managed KMS key that does not grant GuardDuty access."
  resolution: "Update the KMS key policy to grant the GuardDuty service-linked role kms:CreateGrant, kms:Decrypt, kms:GenerateDataKeyWithoutPlaintext."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix KMS key policy | YELLOW | Permission change; reversible |
| Retry scan | YELLOW | Creates EBS snapshot; snapshot can be deleted |
| Configure scan exclusions | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Detector configuration changes
- KMS key policy modifications
- Persistent scan failures

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-malware-scans` (failed) | MEDIUM | Scan failure details |
| `describe-volumes` | MEDIUM | Volume configuration |
| `describe-key` (KMS) | MEDIUM | Key policy details |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| KMS key policy change | Revert key policy to previous version |
| Scan exclusion addition | Remove exclusion tags |

## Output Format

```yaml
root_cause: "scan_failure — <specific_cause>"
evidence:
  - type: scan_details
    content: "<failed scan details and reason>"
  - type: volume
    content: "<EBS volume configuration>"
  - type: kms_key
    content: "<KMS key policy if applicable>"
severity: HIGH
mitigation:
  immediate: "Fix the failure cause and retry the scan"
  long_term: "Ensure KMS policies and volume configurations support scanning"
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
