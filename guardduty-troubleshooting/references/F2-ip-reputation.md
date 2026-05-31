---
title: "F2 — IP Reputation and Trusted IP Lists"
description: "Diagnose GuardDuty trusted IP list and IP reputation issues"
status: active
severity: MEDIUM
triggers:
  - "trusted IP"
  - "IP whitelist"
  - "IP reputation"
  - "trusted list"
  - "IP allowlist"
owner: devops-agent
objective: "Identify and resolve trusted IP list configuration issues"
context: "Trusted IP lists prevent GuardDuty from generating findings for traffic from specified IPs. They only affect VPC Flow Log and CloudTrail-based findings — NOT DNS, malware, or runtime findings. Maximum 6 IP sets per detector, 250,000 IPs per list."
---

## Phase 1 — Triage

MUST:
- List IP sets: `aws guardduty list-ip-sets --detector-id <detector-id>`
- Get IP set details: `aws guardduty get-ip-set --detector-id <detector-id> --ip-set-id <set-id>`
- Check the set status (ACTIVE, INACTIVE, ERROR)
- Verify the S3 location is accessible: `aws s3 ls <s3-location>`
- Confirm the file contains valid CIDR notation IPs

SHOULD:
- Verify the S3 bucket policy allows GuardDuty access
- Check if the trusted IPs are correctly formatted (one per line, CIDR notation)
- Verify the detector is ENABLED

MAY:
- Check if findings are still generated for trusted IPs (indicates the list is not working)
- Verify the IP set does not exceed 250,000 entries

## Phase 2 — Remediate

MUST:
- Fix S3 access issues for the IP list file
- Ensure IPs are in valid CIDR notation (e.g., 10.0.0.1/32)
- Activate the IP set if inactive: `aws guardduty update-ip-set --detector-id <detector-id> --ip-set-id <set-id> --activate`

SHOULD:
- Review trusted IPs periodically to remove stale entries
- Document the justification for each trusted IP range

MAY:
- Automate trusted IP list updates
- Set up monitoring for IP set status changes

## Common Issues

- symptoms: "Trusted IP list active but findings still generated for listed IPs"
  diagnosis: "Trusted IP lists only suppress VPC Flow Log and CloudTrail-based findings. DNS, malware, and runtime findings are not affected."
  resolution: "Use suppression rules for finding types not covered by trusted IP lists."

- symptoms: "IP set shows ERROR status"
  diagnosis: "The S3 file cannot be read or contains invalid IP format."
  resolution: "Verify S3 access and ensure IPs are in CIDR notation (e.g., 192.168.1.0/24), one per line."

- symptoms: "Cannot create IP set — limit exceeded"
  diagnosis: "Maximum 6 IP sets per detector."
  resolution: "Consolidate existing IP sets or delete unused ones."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix S3 access for IP list | YELLOW | Permission change; reversible |
| Activate IP set | YELLOW | Configuration change; reversible |
| Update IP list file | YELLOW | Content change; reversible |

## Escalation Conditions

- Trusted IP list modifications (may hide real threats)
- Detector configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-ip-set` | MEDIUM | Trusted IP list location and status |
| `list-ip-sets` | LOW | IP set IDs |
| S3 list file contents | MEDIUM | Trusted IP addresses |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection
- NEVER add overly broad CIDR ranges to trusted IP lists

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| IP set activation | Deactivate via `update-ip-set` |
| IP list file update | Revert S3 file to previous version |

## Output Format

```yaml
root_cause: "ip_reputation — <specific_cause>"
evidence:
  - type: ip_set
    content: "<IP set details and status>"
  - type: s3_access
    content: "<S3 file accessibility>"
severity: MEDIUM
mitigation:
  immediate: "Fix the IP set configuration and activate"
  long_term: "Review trusted IPs periodically and document justifications"
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
