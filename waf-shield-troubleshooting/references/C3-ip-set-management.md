---
title: "C3 — IP Set Management Issues"
description: "Diagnose IP set size limits, CIDR format errors, and management problems"
status: active
severity: MEDIUM
triggers:
  - "IP set"
  - "IP set limit"
  - "CIDR format"
  - "IP set update"
  - "too many IPs"
owner: devops-agent
objective: "Resolve IP set management issues including size limits, format errors, and update conflicts"
context: "WAF IP sets store lists of IP addresses in CIDR notation. Each set supports up to 10,000 CIDR ranges. Sets are either IPv4 or IPv6 (not both). Updates require a lock token for optimistic concurrency. IP sets are scoped (REGIONAL or CLOUDFRONT) and must match the web ACL scope."
---

## Phase 1 — Triage

MUST:
- List IP sets: `aws wafv2 list-ip-sets --scope <scope>`
- Get IP set details and current lock token: `aws wafv2 get-ip-set --name <ip-set-name> --scope <scope> --id <ip-set-id>`
- Check the number of addresses in the set
- Verify the IP address version (IPV4 or IPV6)
- Check CIDR notation format for all addresses

SHOULD:
- Verify the IP set scope matches the web ACL scope
- Check for duplicate or overlapping CIDR ranges
- Verify the lock token is current (concurrent updates cause conflicts)

MAY:
- Audit IP sets for stale entries that can be removed
- Check if IP sets are shared across multiple rules or web ACLs

## Phase 2 — Remediate

MUST:
- Use correct CIDR notation: /32 for single IPv4, /128 for single IPv6
- Include the current lock token in update calls: `aws wafv2 update-ip-set --name <ip-set-name> --scope <scope> --id <ip-set-id> --addresses <cidr-list> --lock-token <lock-token>`
- Stay within the 10,000 address limit per IP set

SHOULD:
- Consolidate individual IPs into larger CIDR blocks where possible
- Use separate IP sets for different purposes (allow list, block list, rate limit exceptions)
- Implement automation for IP set updates with retry logic for lock token conflicts

MAY:
- Use AWS Firewall Manager for cross-account IP set management
- Implement IP set rotation for threat intelligence feeds
- Create Lambda functions to automatically update IP sets from external feeds

## Common Issues

- symptoms: "Update IP set fails with WAFOptimisticLockException"
  diagnosis: "Another process updated the IP set between your get and update calls."
  resolution: "Re-fetch the IP set to get the current lock token and retry the update."

- symptoms: "Cannot add IP — set at 10,000 limit"
  diagnosis: "IP set has reached the maximum 10,000 CIDR range limit."
  resolution: "Consolidate IPs into larger CIDR blocks. Create additional IP sets with separate rules."

- symptoms: "IP set update fails with invalid CIDR"
  diagnosis: "IP address format is incorrect (missing prefix length, wrong notation)."
  resolution: "Use /32 for single IPv4 (e.g., 192.168.1.1/32) and /128 for single IPv6."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Update IP set addresses | YELLOW | IP list change; reversible |
| Consolidate CIDR blocks | YELLOW | IP list restructure; reversible |
| Create additional IP sets | YELLOW | New resource; can be deleted |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Large-scale IP set modifications

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-ip-set` | MEDIUM | IP addresses |
| `list-ip-sets` | LOW | IP set names and IDs |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER clear an entire IP set without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| IP set update | Revert addresses via `update-ip-set` with previous list |
| CIDR consolidation | Revert to original individual IPs |
| New IP set creation | Delete IP set via `delete-ip-set` |

## Output Format

```yaml
root_cause: "ip_set_management — <specific_cause>"
evidence:
  - type: ip_set_size
    content: "<current address count>"
  - type: ip_version
    content: "<IPV4 or IPV6>"
  - type: error_message
    content: "<API error if applicable>"
severity: MEDIUM
mitigation:
  immediate: "Fix CIDR format or resolve lock token conflict"
  long_term: "Implement automated IP set management with consolidation"
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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"
