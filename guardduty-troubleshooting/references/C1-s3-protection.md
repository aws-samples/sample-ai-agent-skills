---
title: "C1 — S3 Protection Issues"
description: "Diagnose GuardDuty S3 protection configuration and detection issues"
status: active
severity: HIGH
triggers:
  - "S3 protection"
  - "S3 findings missing"
  - "S3 threat detection"
  - "S3 data events"
  - "S3 suspicious access"
owner: devops-agent
objective: "Identify and resolve S3 protection issues preventing detection of S3-related threats"
context: "GuardDuty S3 Protection monitors CloudTrail S3 data events to detect suspicious S3 API activity. It detects anomalous data access patterns, S3 bucket policy changes, and data exfiltration attempts. S3 protection uses its own data stream independent of CloudTrail S3 data event logging."
---

## Phase 1 — Triage

MUST:
- Check S3 protection status: `aws guardduty get-detector --detector-id <detector-id>` (check S3_DATA_EVENTS feature)
- List S3-related findings: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"type":{"Eq":["Policy:S3/BucketAnonymousAccessGranted","Exfiltration:S3/MaliciousIPCaller"]}}}' --max-results 10`
- Verify the detector is ENABLED
- Check if the account has S3 buckets with activity

SHOULD:
- Verify the service-linked role permissions for S3 data event access
- Check for S3-related findings in the last 30 days
- Review CloudTrail for S3 data event volume

MAY:
- Check S3 bucket policies for known risky configurations
- Verify S3 access logging is enabled for correlation

## Phase 2 — Remediate

MUST:
- Enable S3 protection if disabled: `aws guardduty update-detector --detector-id <detector-id> --features '[{"Name":"S3_DATA_EVENTS","Status":"ENABLED"}]'`
- Verify S3 protection status changed to ENABLED

SHOULD:
- Enable S3 protection across all member accounts via organization configuration
- Monitor S3 finding generation after enabling

MAY:
- Set up EventBridge rules for automated S3 finding response
- Create suppression rules for expected S3 access patterns

## Common Issues

- symptoms: "No S3-related GuardDuty findings despite suspicious S3 activity"
  diagnosis: "S3 protection is not enabled in the detector configuration."
  resolution: "Enable S3_DATA_EVENTS feature in the detector."

- symptoms: "S3 protection enabled but only seeing policy findings, not data access findings"
  diagnosis: "Data access findings require sufficient S3 API activity to establish a baseline for anomaly detection."
  resolution: "Allow time for GuardDuty to establish a baseline (7-14 days). Anomaly-based findings need historical data."

- symptoms: "S3 protection costs higher than expected"
  diagnosis: "High volume of S3 data events being analyzed, possibly from automated processes."
  resolution: "Review S3 data event volume. Consider if all buckets need monitoring or if some can be excluded."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable S3 protection | YELLOW | Configuration change; reversible |
| Enable across member accounts | YELLOW | Org-wide change; reversible |

## Escalation Conditions

- Detector configuration changes
- S3 protection enablement/disablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` (S3 feature) | LOW | Feature status |
| `list-findings` (S3 type) | HIGH | S3 threat details |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| S3 protection enablement | Disable via `update-detector` (not recommended) |

## Output Format

```yaml
root_cause: "s3_protection — <specific_cause>"
evidence:
  - type: detector_features
    content: "<S3 protection status>"
  - type: findings
    content: "<S3-related findings>"
severity: HIGH
mitigation:
  immediate: "Enable S3 protection and verify finding generation"
  long_term: "Monitor S3 protection costs and finding patterns"
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
