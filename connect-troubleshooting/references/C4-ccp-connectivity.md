---
title: "C4 — CCP Connectivity Issues"
description: "Diagnose Contact Control Panel connectivity and WebRTC problems"
status: active
severity: HIGH
triggers:
  - "CCP not loading"
  - "No audio"
  - "One-way audio"
  - "CCP disconnects"
  - "WebRTC error"
owner: devops-agent
objective: "Restore CCP connectivity and audio quality"
context: "CCP uses WebRTC for real-time audio. It requires specific network access (ports, domains), a supported browser, and stable network connectivity. Corporate firewalls, VPNs, and proxy servers are the most common causes of CCP issues."
---

## Phase 1 — Triage

MUST:
- Check browser compatibility (Chrome or Firefox latest versions)
- Verify network access to Connect domains (*.awsapps.com, *.amazonaws.com, *.cloudfront.net)
- Check if ports 3478 UDP/TCP (TURN/STUN) are open
- Check if the agent is on VPN (common cause of audio issues)

SHOULD:
- Run the Connect endpoint connectivity test tool
- Check for proxy servers intercepting WebRTC traffic
- Verify the agent has only one CCP tab open
- Check browser console for WebRTC errors

MAY:
- Check network quality metrics (jitter, packet loss, latency)
- Test from a different network to isolate the issue
- Check if the issue affects all agents or specific locations

## Phase 2 — Remediate

MUST:
- Whitelist required Connect domains and ports in the firewall
- If VPN: configure split tunneling to exclude Connect media traffic
- Use a supported browser (Chrome or Firefox)

SHOULD:
- Disable proxy for Connect domains
- Ensure WebRTC is not blocked by browser extensions
- Close other CCP sessions (only one per agent)

MAY:
- Implement network quality monitoring for agent locations
- Set up a dedicated network path for Connect traffic
- Deploy the Connect client application instead of browser CCP

## Common Issues

- symptoms: "CCP loads but no audio (one-way or no-way)"
  diagnosis: "WebRTC media ports blocked by firewall or VPN."
  resolution: "Open port 3478 UDP/TCP. Configure VPN split tunneling for Connect traffic."

- symptoms: "CCP disconnects intermittently"
  diagnosis: "Unstable network connection or WebSocket timeout."
  resolution: "Check network stability. Ensure WebSocket connections aren't being terminated by proxy/firewall."

- symptoms: "Choppy audio or echo"
  diagnosis: "Network jitter > 30ms or packet loss > 0.5%."
  resolution: "Improve network quality. Use wired connection instead of WiFi. Check for bandwidth contention."

- symptoms: "CCP shows 'Initialization failed'"
  diagnosis: "Browser cannot reach Connect domains or cookies are blocked."
  resolution: "Whitelist Connect domains. Enable third-party cookies for Connect domain."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Network configuration changes: YELLOW - May affect connectivity"
```

## Escalation Conditions

- Contact center is live with active calls or chats in progress
- Issue affects agent ability to receive or handle contacts
- Fix requires modifying a contact flow associated with a phone number
- Resolution involves changes to hours of operation during business hours
- Issue impacts call recording, CTR generation, or compliance requirements

## Data Sensitivity

HIGH - Contact Trace Records (CTRs) contain customer phone numbers, call timestamps, and interaction details. Call recordings contain customer voice data. Chat transcripts may contain PII, financial data, or healthcare information. Agent data includes employee performance metrics.

## Prohibited Actions

- NEVER suggest deleting a contact flow that is associated with a phone number - this breaks inbound routing
- NEVER suggest modifying production hours of operation during active business hours
- NEVER recommend removing security profiles from agents while they are logged in
- NEVER suggest disabling call recording on queues that have compliance requirements
- NEVER recommend deleting or modifying a published contact flow without first testing in a non-production flow

## Phase 3 - Rollback

1. If agent routing or security profiles were modified, restore original profile assignments
2. If phone number associations were changed, reassociate the number with the original contact flow
## Output Format

```yaml
root_cause: "ccp_connectivity — <specific issue>"
evidence:
  - type: network_diagnostics
    content: "<connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Fix network/firewall configuration"
  long_term: "Implement network requirements documentation and monitoring"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
