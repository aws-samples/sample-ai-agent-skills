---
title: "G1 — Security Hub Export"
description: "Diagnose GuardDuty finding export to Security Hub issues"
status: active
severity: MEDIUM
triggers:
  - "security hub export"
  - "findings not in security hub"
  - "guardduty security hub"
  - "finding integration"
  - "security hub findings missing"
owner: devops-agent
objective: "Identify and resolve GuardDuty finding export issues to Security Hub"
context: "GuardDuty automatically sends findings to Security Hub when both services are enabled in the same region. Issues arise when Security Hub is not enabled, the integration is disabled, or findings are not appearing in Security Hub due to format or timing issues."
---

## Phase 1 — Triage

MUST:
- Verify GuardDuty is enabled: `aws guardduty get-detector --detector-id <detector-id>`
- Verify Security Hub is enabled: `aws securityhub describe-hub`
- Check GuardDuty integration in Security Hub: `aws securityhub list-enabled-products-for-import`
- Check for GuardDuty findings in Security Hub: `aws securityhub get-findings --filters '{"ProductName":[{"Value":"GuardDuty","Comparison":"EQUALS"}]}' --max-results 10`
- Verify both services are in the same region

SHOULD:
- Check publishing destinations: `aws guardduty list-publishing-destinations --detector-id <detector-id>`
- Verify the GuardDuty product integration is enabled in Security Hub
- Check for recent GuardDuty findings: `aws guardduty list-findings --detector-id <detector-id> --max-results 5`

MAY:
- Check CloudTrail for Security Hub integration events
- Verify cross-region aggregation if using centralized Security Hub

## Phase 2 — Remediate

MUST:
- Enable Security Hub if not enabled: `aws securityhub enable-security-hub`
- Enable the GuardDuty product integration in Security Hub
- Verify findings start appearing in Security Hub

SHOULD:
- Set up cross-region aggregation in Security Hub for centralized view
- Configure Security Hub custom actions for GuardDuty findings

MAY:
- Set up S3 export as a backup publishing destination
- Create EventBridge rules for finding notifications

## Common Issues

- symptoms: "GuardDuty findings not appearing in Security Hub"
  diagnosis: "Security Hub is not enabled in the same region, or the GuardDuty integration is disabled."
  resolution: "Enable Security Hub and verify the GuardDuty product integration is active."

- symptoms: "Only some GuardDuty findings appear in Security Hub"
  diagnosis: "Suppressed (archived) GuardDuty findings are not sent to Security Hub."
  resolution: "Suppressed findings are excluded from Security Hub export. Review suppression rules if needed."

- symptoms: "GuardDuty findings in Security Hub have stale status"
  diagnosis: "Finding updates in GuardDuty may take time to propagate to Security Hub."
  resolution: "Allow time for propagation. Finding status updates are eventually consistent."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable Security Hub | YELLOW | Service enablement; reversible |
| Enable GuardDuty integration | YELLOW | Configuration change; reversible |
| Set up S3 export | YELLOW | New resource; can be deleted |

## Escalation Conditions

- Detector configuration changes
- Security Hub enablement/configuration

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` | MEDIUM | Detector configuration |
| `describe-hub` | LOW | Security Hub status |
| `get-findings` (Security Hub) | HIGH | Finding details |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Security Hub enablement | Disable via `disable-security-hub` (not recommended) |
| Integration enablement | Disable product integration |

## Output Format

```yaml
root_cause: "security_hub_export — <specific_cause>"
evidence:
  - type: guardduty_status
    content: "<detector status>"
  - type: security_hub
    content: "<Security Hub status and integration>"
  - type: findings
    content: "<finding presence in Security Hub>"
severity: MEDIUM
mitigation:
  immediate: "Enable Security Hub and GuardDuty integration"
  long_term: "Set up cross-region aggregation and automated response"
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
