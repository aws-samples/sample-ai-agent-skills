---
title: "C1 — IP Blocking/Allowing Issues"
description: "Diagnose IP-based rules not blocking or allowing traffic as expected"
status: active
severity: HIGH
triggers:
  - "IP block"
  - "IP allow"
  - "IP whitelist"
  - "IP blacklist"
  - "IP not blocked"
  - "IP still allowed"
owner: devops-agent
objective: "Identify why IP-based WAF rules are not working as expected and fix the configuration"
context: "WAF IP set rules match requests based on source IP address. IP sets support IPv4 and IPv6 in CIDR notation. Common issues include incorrect CIDR notation, wrong IP version, clients behind proxies (X-Forwarded-For), IP set scope mismatch, and rule priority conflicts. IP sets have a 10,000 address limit per set."
---

## Phase 1 — Triage

MUST:
- Get the IP set configuration: `aws wafv2 get-ip-set --name <ip-set-name> --scope <scope> --id <ip-set-id>`
- Verify the IP address is in the set with correct CIDR notation (use /32 for single IPv4, /128 for single IPv6)
- Check the rule using the IP set and its priority: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id>`
- Verify the rule action (Block for blocking, Allow for allowing)
- Check if the client IP is the actual source IP or if it's behind a proxy/CDN

SHOULD:
- Check if the IP set is IPv4 or IPv6 and matches the client's IP version
- Verify the IP set scope matches the web ACL scope (REGIONAL or CLOUDFRONT)
- Check if a higher-priority rule is matching before the IP rule

MAY:
- Check WAF logs to see the source IP WAF is evaluating
- Test with a known IP: `curl -v --interface <ip> https://<domain>/`

## Phase 2 — Remediate

MUST:
- Add the correct IP in proper CIDR notation: `aws wafv2 update-ip-set --name <ip-set-name> --scope <scope> --id <ip-set-id> --addresses <cidr-list> --lock-token <lock-token>`
- Use IPSetForwardedIPConfig if clients are behind a proxy: configure the header name (X-Forwarded-For) and fallback behavior
- Ensure the IP rule priority is correct relative to other rules

SHOULD:
- Use separate IP sets for allow lists and block lists
- Document the purpose of each IP in the set for maintenance
- Set up automation to update IP sets (e.g., threat intelligence feeds)

MAY:
- Use AWS Firewall Manager to manage IP sets across multiple accounts
- Implement IP set rotation for dynamic threat lists

## Common Issues

- symptoms: "IP is in the block list but traffic from that IP is not blocked"
  diagnosis: "Client is behind a CDN/proxy. WAF sees the proxy IP, not the client IP."
  resolution: "Use IPSetForwardedIPConfig with X-Forwarded-For header."

- symptoms: "IP allow list not working — trusted IP still blocked"
  diagnosis: "Allow rule priority is higher (larger number) than the Block rule. Block evaluates first."
  resolution: "Set the Allow rule to a lower priority number than the Block rule."

- symptoms: "Cannot add more IPs to the set"
  diagnosis: "IP set has reached the 10,000 address limit."
  resolution: "Consolidate IPs into larger CIDR blocks or create additional IP sets with separate rules."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Update IP set addresses | YELLOW | IP list change; reversible |
| Configure ForwardedIPConfig | YELLOW | Rule modification; reversible |
| Adjust rule priority | YELLOW | Evaluation order change; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- IP allow/block list changes affecting production traffic

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-ip-set` | MEDIUM | IP addresses in allow/block lists |
| `get-web-acl` | LOW | Rule configuration |
| WAF logs (source IP) | MEDIUM | Client IP addresses |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting
- NEVER clear an entire IP block list without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| IP set update | Revert addresses via `update-ip-set` with previous list |
| ForwardedIPConfig change | Revert via `update-web-acl` |
| Priority change | Revert priority via `update-web-acl` |

## Output Format

```yaml
root_cause: "ip_blocking — <specific_cause>"
evidence:
  - type: ip_set
    content: "<IP set contents and type>"
  - type: rule_config
    content: "<rule action and priority>"
  - type: source_ip
    content: "<actual IP WAF evaluates>"
severity: HIGH
mitigation:
  immediate: "Fix IP set contents or forwarded IP configuration"
  long_term: "Implement proper IP management with forwarded IP support"
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
