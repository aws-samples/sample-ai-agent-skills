---
title: "A2 — False Positive Management"
description: "Identify and manage false positive GuardDuty findings"
status: active
severity: MEDIUM
triggers:
  - "false positive"
  - "not a real threat"
  - "legitimate activity"
  - "expected behavior flagged"
  - "benign finding"
owner: devops-agent
objective: "Properly identify false positive findings and configure appropriate suppression without missing real threats"
context: "False positives occur when legitimate activity matches GuardDuty detection patterns. Common causes include penetration testing, security scanning tools, expected cross-account access, or known third-party service IPs. Proper false positive management requires careful validation before suppression."
---

## Phase 1 — Triage

MUST:
- Get the full finding details: `aws guardduty get-findings --detector-id <detector-id> --finding-ids <finding-id>`
- Verify the activity is genuinely expected and authorized
- Check the actor details (IP, user agent, account) against known legitimate sources
- Confirm the resource owner acknowledges the activity as expected
- Review the finding type to understand what was detected

SHOULD:
- Check if the source IP belongs to a known service or partner: verify against internal IP registries
- Verify if the activity correlates with scheduled operations (pen testing, vulnerability scanning)
- Check for similar findings that may indicate a pattern of false positives
- Review trusted IP lists: `aws guardduty list-ip-sets --detector-id <detector-id>`

MAY:
- Check the finding history for the same resource to see if this is recurring
- Verify with the security team that the activity was authorized

## Phase 2 — Remediate

MUST:
- Only suppress after confirming the activity is genuinely legitimate
- Create a targeted suppression rule with specific criteria (not overly broad)
- Document the justification for the false positive determination

SHOULD:
- Add known legitimate IPs to trusted IP lists (for network/API findings only): `aws guardduty create-ip-set --detector-id <detector-id> --name <name> --format TXT --location s3://<bucket>/<key> --activate`
- Create suppression rules with narrow criteria: `aws guardduty create-filter --detector-id <detector-id> --name <filter-name> --action ARCHIVE --finding-criteria '<criteria>'`
- Review existing suppression rules periodically

MAY:
- Set up a review process for suppressed findings
- Create a dashboard for tracking false positive rates

## Common Issues

- symptoms: "Repeated findings for authorized penetration testing"
  diagnosis: "Pen testing tools trigger GuardDuty findings for expected scanning and exploitation attempts."
  resolution: "Add pen testing IPs to trusted IP list or create suppression rules for the test period. Remove after testing."

- symptoms: "Findings for expected cross-account API calls"
  diagnosis: "Legitimate cross-account access patterns trigger UnauthorizedAccess findings."
  resolution: "Create a suppression rule matching the specific account ID and API actions."

- symptoms: "DNS findings for legitimate third-party service domains"
  diagnosis: "DNS queries to domains that match threat intelligence patterns but are legitimate services."
  resolution: "Create a suppression rule for the specific finding type and domain pattern."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add IPs to trusted IP list | YELLOW | Suppresses findings for those IPs; reversible |
| Create suppression rule | YELLOW | Auto-archives matching findings; reversible |
| Archive false positive findings | YELLOW | State change; can be unarchived |

## Escalation Conditions

- Suppression rule creation (all suppression rules require review)
- Trusted IP list modifications
- Detector configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-findings` | HIGH | Threat details, resource IDs, actor IPs |
| `list-ip-sets` | MEDIUM | Trusted IP addresses |
| `list-filters` | LOW | Suppression rule names |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection
- NEVER create overly broad suppression rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Trusted IP list addition | Remove IPs via `update-ip-set` |
| Suppression rule creation | Delete filter via `delete-filter` |
| Finding archival | Unarchive via `unarchive-findings` |

## Output Format

```yaml
root_cause: "false_positive — <finding_type>"
evidence:
  - type: finding
    content: "<finding details>"
  - type: validation
    content: "<evidence confirming legitimate activity>"
  - type: suppression
    content: "<suppression rule created>"
severity: MEDIUM
mitigation:
  immediate: "Create targeted suppression rule for the confirmed false positive"
  long_term: "Maintain trusted IP lists and review suppression rules periodically"
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
