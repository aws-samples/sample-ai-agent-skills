---
title: "B1 — Detector Configuration"
description: "Diagnose GuardDuty detector configuration and status issues"
status: active
severity: CRITICAL
triggers:
  - "detector not enabled"
  - "detector configuration"
  - "guardduty disabled"
  - "detector status"
  - "detector suspended"
owner: devops-agent
objective: "Identify and resolve GuardDuty detector configuration issues"
context: "The detector is the foundation of GuardDuty. Without an active detector, no findings are generated. Issues include disabled detectors, suspended detectors, incorrect feature configuration, and missing detectors in specific regions."
---

## Phase 1 — Triage

MUST:
- List detectors: `aws guardduty list-detectors`
- Get detector details: `aws guardduty get-detector --detector-id <detector-id>`
- Check detector status (ENABLED or DISABLED)
- Verify feature configuration (S3 protection, EKS, Lambda, RDS, Malware, Runtime)
- Check the finding publishing frequency setting

SHOULD:
- Verify the detector exists in the expected region
- Check if the detector is managed by a delegated admin: `aws guardduty get-administrator-account --detector-id <detector-id>`
- Review CloudTrail for detector changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateDetector --max-results 10`

MAY:
- Check all regions for detector status
- Verify the service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForAmazonGuardDuty`

## Phase 2 — Remediate

MUST:
- Enable the detector if disabled: `aws guardduty create-detector --enable`
- Resume the detector if suspended
- Enable required features in the detector configuration

SHOULD:
- Set finding publishing frequency to 15 minutes for faster detection
- Enable all relevant protection features for the workload
- Verify the service-linked role has not been modified

MAY:
- Enable GuardDuty in all regions for comprehensive coverage
- Set up automated detector enablement for new regions

## Common Issues

- symptoms: "No GuardDuty findings being generated in a region"
  diagnosis: "The detector is not enabled, is suspended, or does not exist in that region."
  resolution: "Create or enable the detector in the target region."

- symptoms: "Detector exists but specific finding types are missing"
  diagnosis: "The relevant protection feature (S3, EKS, Lambda, etc.) is not enabled."
  resolution: "Enable the specific feature in the detector configuration."

- symptoms: "Detector was disabled unexpectedly"
  diagnosis: "Someone or an automation disabled the detector. Check CloudTrail for the DeleteDetector or UpdateDetector event."
  resolution: "Re-enable the detector and investigate who disabled it. Set up CloudWatch alarms for detector changes."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable detector | YELLOW | Enables finding generation; reversible |
| Resume suspended detector | YELLOW | Resumes monitoring; reversible |
| Enable features | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Detector configuration changes
- Feature enablement/disablement
- Detector found disabled unexpectedly (security incident indicator)

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` | MEDIUM | Detector configuration and features |
| `list-detectors` | LOW | Detector IDs |
| CloudTrail events | MEDIUM | Configuration change history |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Detector enablement | Suspend via `update-detector` (not recommended) |
| Feature enablement | Disable feature via `update-detector` |

## Output Format

```yaml
root_cause: "detector_configuration — <specific_cause>"
evidence:
  - type: detector
    content: "<get-detector output>"
  - type: features
    content: "<feature enablement status>"
  - type: cloudtrail
    content: "<detector change events>"
severity: CRITICAL
mitigation:
  immediate: "Enable or resume the detector with required features"
  long_term: "Monitor detector status and prevent unauthorized changes"
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
