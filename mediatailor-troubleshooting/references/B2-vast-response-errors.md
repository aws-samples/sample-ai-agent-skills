---
title: "B2 — VAST Response Errors"
description: "Diagnose VAST/VMAP response parsing and processing errors"
status: active
severity: HIGH
triggers: ["VAST error", "VMAP error", "invalid ad response", "VAST parsing failed", "empty VAST"]
owner: devops-agent
objective: "Resolve VAST/VMAP response errors from ad decision servers"
context: "MediaTailor expects valid VAST 2.0/3.0/4.x or VMAP 1.0 XML from the ADS. Invalid responses cause ad insertion failures."
---

## Phase 1 — Triage

MUST:
- Test ADS URL directly: `curl -s "<ads-url>" | xmllint --format -`
- Check VAST response for required elements (Ad, InLine/Wrapper, MediaFile)
- Verify VAST version in response matches supported versions (2.0, 3.0, 4.x)

SHOULD:
- Check for VAST wrapper/redirect chains and their depth
- Verify MediaFile URLs in VAST response are accessible
- Check CloudWatch Logs for VAST parsing errors (if logging enabled)

MAY:
- Use a VAST validator tool to check response compliance
- Compare VAST response with IAB VAST specification

## Phase 2 — Remediate

MUST:
- Fix ADS to return valid VAST XML with required elements
- Ensure MediaFile elements contain supported video formats (MP4, TS)
- Resolve VAST wrapper redirect chains that exceed depth limits

SHOULD:
- Add error handling for empty VAST responses (configure slate)
- Implement VAST response caching on ADS side to reduce errors

MAY:
- Configure VAST timeout settings if ADS is slow to respond
- Set up ADS fallback URLs for redundancy

## Safety Ratings

```
safety_ratings:
  - "curl ADS URL: GREEN — read-only HTTP request"
  - "xmllint validation: GREEN — local XML parsing"
  - "CloudWatch Logs query: GREEN — read-only log access"
  - "put-playback-configuration: YELLOW — updates ADS URL affecting future sessions"
```

## Escalation Conditions

- ADS consistently returns invalid VAST requiring ad server vendor intervention
- VAST wrapper chains exceed 5 redirects indicating ad network misconfiguration
- MediaFile URLs are blocked by CDN or firewall requiring network changes
- VAST response times exceed 5 seconds causing viewer experience degradation

## Data Sensitivity

- VAST responses: MEDIUM — contain ad creative URLs and tracking pixels
- ADS request URLs: HIGH — may contain viewer targeting parameters and session tokens
- MediaFile URLs: LOW — public ad creative URLs
- Tracking pixel URLs: MEDIUM — contain impression and click tracking endpoints

## Prohibited Actions

- NEVER cache or store VAST responses containing viewer-specific targeting data
- NEVER bypass VAST wrapper chains by directly accessing inner VAST URLs
- NEVER modify VAST responses manually in transit between ADS and MediaTailor
- NEVER expose ADS authentication credentials when sharing VAST debugging information

## Phase 3 — Rollback

1. If ADS URL was changed and new ADS returns worse VAST, revert ADS URL in configuration
2. If VAST timeout was adjusted causing more errors, revert to default timeout
3. If ad creative URLs were updated in ADS and are broken, revert to previous creatives
4. Monitor AdDecisionServer.Errors metric after rollback

## Common Issues

- symptoms: "VAST response is empty XML with no Ad elements"
  diagnosis: "ADS has no ads available for the targeting parameters."
  resolution: "Configure slate URL for empty responses. Review ADS targeting parameters and ad inventory."

- symptoms: "VAST wrapper chain results in timeout"
  diagnosis: "Too many VAST redirects or slow intermediate ad servers."
  resolution: "Reduce wrapper chain depth. Work with ad network to flatten VAST responses."

## Output Format

```yaml
root_cause: "vast_response — <specific_cause>"
evidence:
  - type: vast_error
    content: "<VAST response or error details>"
severity: HIGH
mitigation:
  immediate: "Fix VAST response format"
  long_term: "Implement VAST response validation in ADS"
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
