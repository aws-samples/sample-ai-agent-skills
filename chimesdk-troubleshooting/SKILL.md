---
name: chimesdk-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Chime SDK problems
  by analyzing meeting creation, audio/video issues, messaging, SIP media
  applications, PSTN integration, identity management, and SDK integration
  using structured runbooks.
  Activate when: meeting creation failures, audio/video quality issues,
  messaging problems, SIP media application errors, PSTN audio failures,
  identity/auth issues, SDK integration errors, or the user says something
  is wrong with Chime SDK without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Chime SDK Meetings, Chime SDK
  Messaging, Chime SDK Voice, Chime SDK Identity, CloudWatch, CloudTrail.
---

# Chime SDK Diagnostics

## When to use

Any Amazon Chime SDK investigation — meeting creation, audio/video, messaging, SIP media, PSTN, identity, or SDK integration issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws chime-sdk-meetings list-attendees --meeting-id <meeting-id>
aws chime-sdk-voice list-sip-media-applications --max-results 20
aws chime-sdk-messaging list-channels --app-instance-arn <app-instance-arn>
aws cloudwatch get-metric-statistics --namespace AWS/ChimeSDK --metric-name MeetingCount --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws chime-sdk-meetings get-meeting --meeting-id <meeting-id>
aws chime-sdk-voice get-sip-media-application --sip-media-application-id <id>
aws chime-sdk-identity list-app-instances --max-results 20
aws chime-sdk-messaging describe-channel --channel-arn <channel-arn> --app-instance-user-arn <user-arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chime.amazonaws.com --max-results 10
aws chime-sdk-voice list-phone-numbers --max-results 20
aws chime-sdk-voice get-voice-connector --voice-connector-id <id>
```

Read `references/guardrails.md` before concluding on any Chime SDK issue.

## Gotchas: Chime SDK

- Chime SDK has separate services: Meetings, Messaging, Voice, Identity, Media Pipelines.
- Meeting attendee tokens are sensitive — generate server-side, pass to client.
- SIP media applications require Lambda functions for call control.
- PSTN audio requires phone number provisioning and SIP rules.
- Messaging uses app instances and app instance users for identity.
- Media regions affect latency — choose closest to participants.
- WebRTC is used for meetings — requires UDP connectivity.

## Anti-hallucination rules

1. Chime SDK Meetings, Messaging, and Voice are separate services with different APIs.
2. Meeting IDs and attendee tokens are ephemeral — meetings expire after inactivity.
3. SIP media applications require Lambda for call flow logic.
4. Phone numbers must be provisioned before use in PSTN audio.
5. App instance users are required for messaging — they're not IAM users.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Meetings | A1-A2 | Meeting creation, attendee issues |
| B — Audio/Video | B1-B2 | Audio problems, video quality |
| C — Messaging | C1-C2 | Channel messaging, delivery |
| D — SIP | D1-D2 | SIP media apps, call routing |
| E — PSTN | E1-E2 | Phone numbers, PSTN audio |
| F — Identity | F1 | App instances, user management |
| G — SDK | G1-G2 | Client SDK, integration |
| Z — Catch-All | Z1 | General troubleshooting |
