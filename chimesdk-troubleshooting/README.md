# Chime SDK Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Chime SDK problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Chime SDK — meeting creation, audio/video, messaging, SIP media applications, PSTN integration, identity management, and SDK integration.

### Activate When

- Meeting creation or join failures
- Audio/video quality issues
- Messaging channel or delivery problems
- SIP media application errors
- PSTN audio or phone number issues
- Identity or authentication failures
- Client SDK integration errors

---

## Skill Structure

```
chimesdk-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-meeting-creation.md
    ├── A2-attendee-issues.md
    ├── B1-audio-problems.md
    ├── B2-video-quality.md
    ├── C1-channel-messaging.md
    ├── C2-message-delivery.md
    ├── D1-sip-media-apps.md
    ├── D2-call-routing.md
    ├── E1-phone-numbers.md
    ├── E2-pstn-audio.md
    ├── F1-identity-management.md
    ├── G1-client-sdk.md
    ├── G2-sdk-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `chime-sdk-meetings:*`, `chime-sdk-messaging:*`, `chime-sdk-voice:*`, `chime-sdk-identity:*`

---

## License

MIT-0
