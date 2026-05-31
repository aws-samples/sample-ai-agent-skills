---
title: "B3 — Feature Enablement"
description: "Diagnose GuardDuty feature enablement and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "feature not enabled"
  - "enable feature"
  - "feature configuration"
  - "protection feature"
  - "feature status"
owner: devops-agent
objective: "Identify and resolve GuardDuty feature enablement issues across detector and organization settings"
context: "GuardDuty features (S3 Protection, EKS Protection, Lambda Protection, RDS Protection, Malware Protection, Runtime Monitoring) must be individually enabled. In multi-account setups, features can be managed by the delegated admin. Feature enablement affects which finding types are generated."
---

## Phase 1 — Triage

MUST:
- Get detector with feature status: `aws guardduty get-detector --detector-id <detector-id>`
- Check each feature's status in the detector response (ENABLED/DISABLED)
- Verify organization configuration if managed: `aws guardduty describe-organization-configuration --detector-id <detector-id>`
- Check if the account is a member: `aws guardduty get-administrator-account --detector-id <detector-id>`

SHOULD:
- Check member detector features: `aws guardduty get-member-detectors --detector-id <detector-id> --account-ids <account-id>`
- Review CloudTrail for feature changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateDetector --max-results 10`

MAY:
- Check if auto-enable is configured for new member accounts
- Verify feature availability in the current region

## Phase 2 — Remediate

MUST:
- Enable the required features: `aws guardduty update-detector --detector-id <detector-id> --features '[{"Name":"<feature>","Status":"ENABLED"}]'`
- Verify the feature status changed to ENABLED
- For organization, update organization configuration to auto-enable features

SHOULD:
- Enable features across all member accounts if using delegated admin
- Document which features are enabled and the business justification

MAY:
- Set up monitoring for feature status changes
- Create automation to ensure features remain enabled

## Common Issues

- symptoms: "Feature shows ENABLED but no corresponding findings"
  diagnosis: "The feature is enabled but the account lacks the corresponding resources (no EKS clusters, no Lambda functions, etc.)."
  resolution: "Verify the account has resources that the feature monitors. Some features need time to generate initial findings."

- symptoms: "Cannot enable feature — member account restriction"
  diagnosis: "The account is managed by a delegated admin who controls feature settings."
  resolution: "Contact the delegated admin to enable the feature, or update organization configuration."

- symptoms: "Feature was disabled unexpectedly"
  diagnosis: "The delegated admin changed the configuration, or an automation modified the detector."
  resolution: "Check CloudTrail for UpdateDetector events. Re-enable the feature and prevent unauthorized changes."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable features | YELLOW | Configuration change; reversible |
| Update organization configuration | YELLOW | Org-wide change; reversible |

## Escalation Conditions

- Detector configuration changes
- Organization-wide feature enablement/disablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` (features) | MEDIUM | Feature configuration |
| `describe-organization-configuration` | MEDIUM | Org-wide settings |
| `get-member-detectors` | MEDIUM | Member account feature status |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Feature enablement | Disable feature via `update-detector` (not recommended) |
| Organization config change | Revert via `update-organization-configuration` |

## Output Format

```yaml
root_cause: "feature_enablement — <specific_cause>"
evidence:
  - type: detector_features
    content: "<feature status from get-detector>"
  - type: org_config
    content: "<organization configuration>"
severity: MEDIUM
mitigation:
  immediate: "Enable the required features"
  long_term: "Set up auto-enable for organization and monitor feature status"
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
