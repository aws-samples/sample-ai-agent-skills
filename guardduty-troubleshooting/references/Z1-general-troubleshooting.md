---
title: "Z1 — General GuardDuty Troubleshooting (Catch-All)"
description: "Fallback SOP for GuardDuty issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown GuardDuty issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List detectors: `aws guardduty list-detectors`
- Get detector details: `aws guardduty get-detector --detector-id <detector-id>`
- List recent findings: `aws guardduty list-findings --detector-id <detector-id> --max-results 20`
- Get finding statistics: `aws guardduty get-findings-statistics --detector-id <detector-id> --finding-statistic-types COUNT_BY_SEVERITY`
- Check coverage statistics: `aws guardduty get-coverage-statistics --detector-id <detector-id> --statistics-type COUNT_BY_COVERAGE_STATUS`
- List members: `aws guardduty list-members --detector-id <detector-id>`

SHOULD:
- Check publishing destinations: `aws guardduty list-publishing-destinations --detector-id <detector-id>`
- List threat intel sets: `aws guardduty list-threat-intel-sets --detector-id <detector-id>`
- List IP sets: `aws guardduty list-ip-sets --detector-id <detector-id>`
- Check CloudTrail for GuardDuty events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=guardduty.amazonaws.com --max-results 20`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Finding investigation → Findings (A1-A3)
- Detector or feature issues → Detector (B1-B3)
- Protection feature problems → Protection (C1-C4)
- Malware scanning issues → Malware (D1-D2)
- Multi-account problems → Multi-Account (E1-E2)
- Threat intelligence issues → Threat Intel (F1-F2)
- Integration problems → Integration (G1-G2)
- Runtime monitoring issues → Runtime (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all detector features for their status
- Verify the service-linked role: `aws iam get-role --role-name AWSServiceRoleForAmazonGuardDuty`
- Check organization configuration if applicable: `aws guardduty describe-organization-configuration --detector-id <detector-id>`
- Review recent CloudTrail events for configuration changes

SHOULD:
- Check AWS Health Dashboard for GuardDuty service events
- Verify no SCPs are blocking GuardDuty operations
- Check for recent detector configuration changes
- Review finding export and notification configuration

MAY:
- Check all regions for detector status
- Verify malware scan history
- Review runtime monitoring coverage in detail

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Route to specific SOP | GREEN | Classification only; no state change |
| Investigate and collect evidence | GREEN | Read-only analysis |

## Escalation Conditions

- Detector configuration changes
- Suppression rule creation
- Unclassified issues after investigation

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` | MEDIUM | Detector configuration |
| `list-findings` | MEDIUM | Finding IDs and metadata |
| `get-findings-statistics` | LOW | Aggregate severity counts |
| `list-members` | MEDIUM | Member account IDs |
| CloudTrail events | MEDIUM | API call history |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Route to specific SOP | Follow that SOP's rollback procedures |
| No direct remediation in catch-all | N/A — remediation happens in specific SOPs |

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<findings|detector|protection|malware|multi_account|threat_intel|integration|runtime|unknown>"
investigation_path: "detector → features → findings → coverage → <domain_classification>"
evidence:
  - type: detector
    content: "<detector status and features>"
  - type: findings
    content: "<finding statistics>"
  - type: coverage
    content: "<runtime coverage>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
