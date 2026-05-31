---
title: "E2 — Network Configuration"
description: "Diagnose network configuration issues for Backup Gateway"
status: active
severity: HIGH
triggers: ["network error", "DNS failure", "firewall blocking", "proxy error", "cannot reach AWS"]
owner: devops-agent
objective: "Resolve network configuration issues affecting Backup Gateway operations"
context: "Gateway requires outbound port 443 to AWS endpoints, DNS resolution, NTP, and optionally proxy configuration."
---

## Phase 1 — Triage

MUST:
- Verify gateway can reach AWS endpoints (port 443 outbound)
- Check DNS resolution for AWS service endpoints
- Verify NTP synchronization on the gateway

SHOULD:
- Check proxy configuration if the gateway uses a proxy
- Verify firewall rules allow gateway traffic
- Check for recent network infrastructure changes

MAY:
- Test connectivity to specific AWS endpoints: backup-gateway.<region>.amazonaws.com
- Review network routing tables for the gateway subnet

## Phase 2 — Remediate

MUST:
- Fix DNS configuration to resolve AWS endpoints
- Open firewall rules for port 443 to AWS service endpoints
- Configure NTP for accurate time synchronization

SHOULD:
- Configure proxy settings if direct internet access is not available
- Use static IP for the gateway to simplify firewall rules

MAY:
- Set up AWS PrivateLink endpoints for private connectivity
- Configure redundant DNS servers

## Safety Ratings

```
safety_ratings:
  - "DNS resolution testing: GREEN — read-only network test"
  - "NTP verification: GREEN — read-only time check"
  - "firewall rule changes: RED — modifies network security"
  - "proxy configuration: YELLOW — affects all gateway traffic"
```

## Escalation Conditions

- Firewall changes require network security team approval
- DNS infrastructure changes affect other services
- Proxy configuration requires infrastructure team involvement
- Network routing changes needed for gateway connectivity

## Data Sensitivity

- DNS configuration: LOW — reveals DNS server addresses
- Firewall rules: MEDIUM — reveals network security configuration
- Proxy settings: MEDIUM — reveals proxy infrastructure
- Network topology: MEDIUM — reveals routing and subnet information

## Prohibited Actions

- NEVER open broader firewall rules than required (only port 443 to AWS endpoints)
- NEVER disable the proxy for testing without understanding the security implications
- NEVER modify DNS settings that affect other services
- NEVER skip NTP configuration — time skew causes authentication failures

## Phase 3 — Rollback

1. If firewall rules were modified, revert if connectivity is not restored
2. If DNS was changed, restore original DNS configuration
3. If proxy settings were modified, restore original proxy configuration
4. Verify the gateway reconnects to AWS after network changes
5. Document the network configuration for future reference

## Common Issues

- symptoms: "Gateway cannot reach AWS endpoints after network change"
  diagnosis: "Firewall rules, DNS, or routing changes blocking gateway traffic."
  resolution: "Verify DNS resolves AWS endpoints. Check firewall allows port 443 outbound. Verify routing to AWS endpoints."

## Output Format

```yaml
root_cause: "network_configuration — <specific_cause>"
evidence:
  - type: network_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix network configuration"
  long_term: "Document network requirements and monitor connectivity"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
