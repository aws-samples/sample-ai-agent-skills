---
title: "D3 — Call Quality Issues"
description: "Diagnose audio quality problems including latency, jitter, echo, and packet loss"
status: active
severity: HIGH
triggers:
  - "Poor audio quality"
  - "Choppy audio"
  - "Echo"
  - "Latency"
  - "One-way audio"
owner: devops-agent
objective: "Identify and resolve call quality issues"
context: "Call quality depends on: network path (agent side), WebRTC configuration, headset quality, and PSTN quality (caller side). Agent-side issues are most common and most controllable. Connect provides call quality metrics in CTRs."
---

## Phase 1 — Triage

MUST:
- Check if the issue is agent-side or caller-side (test with different callers)
- Check agent network quality (jitter < 30ms, packet loss < 0.5%, latency < 150ms)
- Verify the agent is using softphone (not desk phone, which adds PSTN latency)
- Check if the agent is on VPN (common cause of quality issues)

SHOULD:
- Review CTR audio quality metrics (if available via Kinesis)
- Check if the issue is consistent or intermittent
- Verify the agent's headset and audio settings
- Check for bandwidth contention on the agent's network

MAY:
- Run network quality tests from the agent's location
- Check if the issue correlates with specific times (peak network usage)
- Review CCP softphone metrics in browser console

## Phase 2 — Remediate

MUST:
- If VPN: configure split tunneling to exclude Connect media traffic
- If network quality poor: improve network path (wired > WiFi, QoS)
- If desk phone: switch to softphone for better quality

SHOULD:
- Implement QoS policies for Connect traffic
- Use wired connections instead of WiFi for agents
- Ensure adequate bandwidth for concurrent agent sessions

MAY:
- Deploy network monitoring for agent locations
- Implement call quality dashboards
- Consider dedicated internet circuits for contact center locations

## Common Issues

- symptoms: "Echo on calls"
  diagnosis: "Agent headset echo cancellation issue or speaker/mic feedback."
  resolution: "Use a headset with echo cancellation. Avoid using speakers."

- symptoms: "One-way audio (agent hears caller but not vice versa)"
  diagnosis: "Firewall blocking outbound UDP media traffic."
  resolution: "Open port 3478 UDP. Configure firewall for WebRTC."

- symptoms: "Choppy audio during peak hours"
  diagnosis: "Network bandwidth contention."
  resolution: "Implement QoS. Ensure sufficient bandwidth. Use wired connections."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
3. If recording or streaming configuration was changed, restore original storage settings
## Output Format

```yaml
root_cause: "call_quality — <specific issue>"
evidence:
  - type: network_metrics
    content: "<jitter, packet loss, latency>"
severity: HIGH
mitigation:
  immediate: "Fix network or audio configuration"
  long_term: "Implement QoS and network monitoring"
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
