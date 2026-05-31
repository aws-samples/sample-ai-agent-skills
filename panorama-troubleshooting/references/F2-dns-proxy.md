---
title: "F2 — DNS and Proxy Configuration"
description: "Diagnose DNS resolution and proxy-related connectivity issues"
status: active
severity: MEDIUM
triggers:
  - "DNS resolution failure"
  - "proxy configuration"
  - "cannot resolve AWS endpoints"
owner: devops-agent
objective: "Resolve DNS and proxy issues affecting Panorama connectivity"
context: "Panorama appliance needs DNS resolution for AWS endpoints. Corporate proxies may interfere with HTTPS connections."
---

## Phase 1 — Triage

MUST:
- Verify DNS server configuration on the device network
- Check if AWS endpoint hostnames resolve correctly
- Verify proxy settings if corporate network requires proxy
- Check device logs for DNS or TLS errors

SHOULD:
- Test DNS resolution for panorama.region.amazonaws.com
- Verify proxy allows HTTPS to AWS endpoints
- Check for certificate inspection that may break TLS

MAY:
- Configure custom DNS servers if default DNS is unreliable
- Test with direct connection (bypass proxy) to isolate issue

## Phase 2 — Remediate

MUST:
- Fix DNS configuration to resolve AWS endpoints
- Configure proxy settings if required by network
- Ensure TLS inspection does not interfere with AWS connections

SHOULD:
- Document proxy and DNS requirements
- Add AWS endpoints to proxy allowlist

## Common Issues

- symptoms: "Device intermittently loses connection"
  diagnosis: "DNS resolution failures or proxy timeouts."
  resolution: "Configure reliable DNS. Add AWS endpoints to proxy allowlist."

## Output Format

```yaml
root_cause: "dns_proxy — <specific_cause>"
evidence:
  - type: network_config
    content: "<DNS and proxy configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix DNS/proxy configuration"
  long_term: "Document network requirements"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — DNS proxy is a network routing function
- Service Disruption Risk: MEDIUM — DNS issues affect service resolution
- Compliance Impact: LOW — DNS proxy is an operational concern

## Escalation Conditions
- DNS proxy fails causing all name resolution to fail
- DNS proxy misconfiguration routes traffic incorrectly
- DNS proxy performance degradation
- DNS proxy security vulnerability detected
- DNS proxy changes affect multiple devices

## Data Sensitivity
- DNS configurations — INTERNAL
- Proxy settings — INTERNAL
- Upstream DNS server addresses — INTERNAL
- DNS query logs — INTERNAL
- Never expose DNS configurations that reveal internal architecture

## Prohibited Actions
- Do NOT configure DNS proxy to use untrusted upstream servers
- Do NOT disable DNS proxy without alternative resolution
- Do NOT bypass DNS proxy security features
- Do NOT ignore DNS resolution failures
- Do NOT modify DNS proxy during active device operations without approval
- Do NOT log DNS queries containing sensitive hostnames

## Phase 3 — Rollback
- Restore previous DNS proxy configurations
- Revert upstream DNS server changes
- Re-enable DNS proxy if disabled
- Verify name resolution after changes
- Restore DNS caching configurations
- Document DNS proxy changes and their impact

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
