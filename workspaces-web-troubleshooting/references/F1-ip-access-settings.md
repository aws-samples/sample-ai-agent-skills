---
title: "F1 — IP Access Settings"
description: "Diagnose WorkSpaces Web IP access restriction issues"
status: active
severity: HIGH
triggers: ["IP blocked", "access denied from IP", "cannot connect from office", "IP restriction error"]
owner: devops-agent
objective: "Resolve IP access settings issues that block or incorrectly allow portal access"
context: "IP access settings restrict which client IP addresses can connect to the portal. Rules are evaluated at the portal endpoint. Clients behind NAT are seen by their public IP. Misconfigured rules can lock out legitimate users."
---

## Phase 1 — Triage

MUST:
- Get IP access settings: `aws workspaces-web get-ip-access-settings --ip-access-settings-arn <arn>`
- Verify the user's public IP address (the IP seen by the portal, not their private IP)
- Check if IP access settings are associated with the portal: `aws workspaces-web get-portal --portal-arn <portal-arn>`

SHOULD:
- Verify the IP ranges in the access rules match the expected client IP ranges
- Check if the user is behind a VPN or proxy that changes their public IP
- Confirm the rule type (ALLOW vs DENY) and evaluation order

MAY:
- Review CloudTrail for recent IP access settings changes
- Check if the issue correlates with a network change (new VPN, office IP change)

## Phase 2 — Remediate

MUST:
- Update IP access rules to include the correct client IP ranges: `aws workspaces-web update-ip-access-settings --ip-access-settings-arn <arn> --ip-rules '[{"ipRange":"<cidr>","description":"Office network"}]'`
- Verify the CIDR notation is correct (e.g., 203.0.113.0/24 not 203.0.113.0/32 for a range)
- Test access from the affected IP after rule changes

SHOULD:
- Include all office, VPN, and remote access IP ranges
- Document the business justification for each IP rule
- Use descriptive names for IP rules

MAY:
- Implement a process for updating IP rules when office IPs change
- Set up monitoring for access denials due to IP restrictions
- Create separate portals for different IP access requirements

## Safety Ratings

```
safety_ratings:
  - "get-ip-access-settings: GREEN — read-only IP rules"
  - "get-portal: GREEN — read-only portal details"
  - "update-ip-access-settings: RED — modifies access control, can lock out users"
  - "create-ip-access-settings: YELLOW — creates access control resource"
  - "associate-ip-access-settings: RED — applies IP restrictions to portal"
```

## Escalation Conditions

- All users locked out due to incorrect IP access rules
- IP access settings cannot be updated due to IAM permission issues
- Dynamic IP ranges (ISP, mobile) make IP-based access control impractical
- Compliance requires IP restrictions but user IPs are unpredictable

## Data Sensitivity

- `aws workspaces-web get-ip-access-settings`: MEDIUM — reveals allowed IP ranges (office locations)
- Client IP addresses: MEDIUM — can identify user locations
- IP access rules: MEDIUM — reveals security access control policy

## Prohibited Actions

- NEVER remove all IP access rules from a production portal without security approval
- NEVER add 0.0.0.0/0 to allow all IPs without explicit security team approval
- NEVER modify IP access settings during business hours without a rollback plan
- NEVER share IP access rules externally — they reveal office network information

## Phase 3 — Rollback

1. Revert IP access rules to previous configuration: `aws workspaces-web update-ip-access-settings --ip-access-settings-arn <arn> --ip-rules '<previous-rules-json>'`
2. If IP access settings were associated in error, disassociate them: `aws workspaces-web disassociate-ip-access-settings --portal-arn <portal-arn>`
3. Delete incorrectly created IP access settings: `aws workspaces-web delete-ip-access-settings --ip-access-settings-arn <arn>`
4. Verify affected users can access the portal after rollback
5. Document the IP access change and rollback reason

## Common Issues

- symptoms: "Users blocked after office network change"
  diagnosis: "Office public IP changed (ISP change, new NAT gateway) and IP access rules have the old IP."
  resolution: "Update IP access rules with the new office public IP range."

## Output Format

```yaml
root_cause: "ip_access_settings — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IP access rules"
  long_term: "Implement IP change notification process and document all allowed ranges"
```

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
