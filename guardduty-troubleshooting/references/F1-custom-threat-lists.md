---
title: "F1 — Custom Threat Intelligence Lists"
description: "Diagnose GuardDuty custom threat intelligence list configuration issues"
status: active
severity: MEDIUM
triggers:
  - "threat intel list"
  - "custom threat list"
  - "threat intelligence"
  - "threat list upload"
  - "threat list not working"
owner: devops-agent
objective: "Identify and resolve custom threat intelligence list configuration and activation issues"
context: "GuardDuty supports custom threat intelligence lists containing known malicious IPs. These lists supplement GuardDuty's built-in threat intelligence. Lists are stored in S3 and must be in the correct format. Maximum 6 threat intel sets per detector."
---

## Phase 1 — Triage

MUST:
- List threat intel sets: `aws guardduty list-threat-intel-sets --detector-id <detector-id>`
- Get threat intel set details: `aws guardduty get-threat-intel-set --detector-id <detector-id> --threat-intel-set-id <set-id>`
- Check the set status (ACTIVE, INACTIVE, ERROR, DELETE_PENDING)
- Verify the S3 location is accessible: `aws s3 ls <s3-location>`
- Check the file format (TXT, STIX, OTX_CSV, ALIEN_VAULT, PROOF_POINT, FIRE_EYE)

SHOULD:
- Verify the S3 bucket policy allows GuardDuty access
- Check if the list file contains valid IP addresses
- Verify the detector is ENABLED

MAY:
- Check CloudTrail for threat intel set events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateThreatIntelSet --max-results 10`
- Verify the list does not exceed size limits

## Phase 2 — Remediate

MUST:
- Fix the S3 access if the list cannot be read
- Ensure the file format matches the specified format type
- Activate the threat intel set if inactive: `aws guardduty update-threat-intel-set --detector-id <detector-id> --threat-intel-set-id <set-id> --activate`

SHOULD:
- Validate IP addresses in the list file
- Update the list regularly with current threat intelligence
- Monitor the set status for errors

MAY:
- Automate threat list updates from external threat feeds
- Set up monitoring for threat intel set status changes

## Common Issues

- symptoms: "Threat intel set shows ERROR status"
  diagnosis: "The S3 file cannot be read, the format is incorrect, or the file contains invalid entries."
  resolution: "Verify S3 access, check file format matches the specified type, and validate IP entries."

- symptoms: "Threat intel set is ACTIVE but no findings generated for listed IPs"
  diagnosis: "The IPs in the list are not communicating with resources in the account, or findings are being suppressed."
  resolution: "Verify the listed IPs are actually communicating with your resources. Check suppression rules."

- symptoms: "Cannot create threat intel set — limit exceeded"
  diagnosis: "Maximum 6 threat intel sets per detector."
  resolution: "Consolidate existing lists or delete unused sets before creating new ones."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix S3 access for list file | YELLOW | Permission change; reversible |
| Activate threat intel set | YELLOW | Configuration change; reversible |
| Update list file | YELLOW | Content change; reversible |

## Escalation Conditions

- Threat intel set creation or modification
- Detector configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-threat-intel-set` | MEDIUM | Threat list location and status |
| `list-threat-intel-sets` | LOW | Set IDs |
| S3 list file contents | MEDIUM | Threat IP addresses |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Threat intel set activation | Deactivate via `update-threat-intel-set` |
| List file update | Revert S3 file to previous version |

## Output Format

```yaml
root_cause: "custom_threat_list — <specific_cause>"
evidence:
  - type: threat_intel_set
    content: "<threat intel set details and status>"
  - type: s3_access
    content: "<S3 file accessibility>"
severity: MEDIUM
mitigation:
  immediate: "Fix the list configuration and activate"
  long_term: "Automate threat list updates and monitor set status"
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
