---
title: "A2 — Real-Time Analytics Errors"
description: "Diagnose Contact Lens real-time analytics issues"
status: active
severity: HIGH
triggers: ["real-time not working", "no live transcript", "real-time analytics missing", "supervisor view empty"]
owner: devops-agent
objective: "Resolve real-time Contact Lens analytics issues during active calls"
context: "Real-time analytics provide live transcription, sentiment, and category detection during active calls. They require Contact Lens enablement and real-time analytics configuration in the contact flow."
---

## Phase 1 — Triage

MUST:
- Verify Contact Lens is enabled (see A1)
- Check real-time segments for an active contact: `aws connect list-realtime-contact-analysis-segments-v2 --instance-id <instance-id> --contact-id <contact-id> --output-type Raw --segment-types Transcript`
- Verify the contact flow has real-time analytics enabled (not just post-contact)

SHOULD:
- Check if the issue affects all agents or specific agents
- Verify the contact is currently active (real-time data is only available during active calls)
- Check CloudWatch metrics for Contact Lens processing

MAY:
- Review the agent's CCP (Contact Control Panel) for real-time display issues
- Check network connectivity between the agent desktop and Connect

## Phase 2 — Remediate

MUST:
- Enable real-time analytics in the contact flow analytics block
- Verify the contact is active before querying real-time APIs
- Ensure the language configured in the analytics block matches the call language

SHOULD:
- Verify IAM permissions include connect:ListRealtimeContactAnalysisSegmentsV2
- Check that the supervisor dashboard is configured to display real-time data

MAY:
- Configure real-time category rules for live alerting
- Set up EventBridge rules for real-time analytics events

## Safety Ratings

```
safety_ratings:
  - "list-realtime-contact-analysis-segments-v2: GREEN — read-only real-time data"
  - "describe-contact: GREEN — read-only contact details"
  - "update contact flow analytics block: RED — modifies live contact flow"
  - "EventBridge rule creation: YELLOW — creates event routing resource"
```

## Escalation Conditions

- Real-time API returns empty results for confirmed active calls with Contact Lens enabled
- Real-time analytics have significant latency (more than 10 seconds behind the conversation)
- Real-time data available via API but not displaying in the supervisor dashboard
- Real-time analytics work for some languages but not others

## Data Sensitivity

- `aws connect list-realtime-contact-analysis-segments-v2`: HIGH — contains live call transcript
- Real-time sentiment data: MEDIUM — reveals customer emotional state
- Real-time category matches: MEDIUM — reveals conversation topics and issues

## Prohibited Actions

- NEVER access real-time transcripts without authorization for call monitoring
- NEVER use real-time APIs for completed contacts — they only work during active calls
- NEVER share real-time transcript data outside authorized monitoring systems
- NEVER modify contact flows during peak call hours without change management

## Phase 3 — Rollback

1. Revert contact flow changes if real-time analytics modification caused routing issues
2. Remove EventBridge rules if they were created in error
3. Verify call routing and recording are functioning after any changes
4. Document the real-time analytics issue and resolution
5. Verify real-time data is flowing for new active contacts

## Common Issues

- symptoms: "Real-time API returns empty for an active contact"
  diagnosis: "Contact flow has post-contact analytics enabled but not real-time analytics."
  resolution: "Enable real-time analytics in the 'Set recording and analytics behavior' block."

## Output Format

```yaml
root_cause: "real_time_analytics — <specific_cause>"
evidence:
  - type: analytics_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix real-time analytics configuration"
  long_term: "Implement real-time analytics monitoring and alerting"
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
