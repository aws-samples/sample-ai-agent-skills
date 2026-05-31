# Z1 — Chime SDK General Troubleshooting

## When No Specific Runbook Matches

## General Diagnostic Commands

```bash
aws chime-sdk-meetings get-meeting --meeting-id <meeting-id>
aws chime-sdk-meetings list-attendees --meeting-id <meeting-id>
aws chime-sdk-voice list-sip-media-applications --max-results 20
aws chime-sdk-voice list-phone-numbers --max-results 20
aws chime-sdk-identity list-app-instances --max-results 20
aws chime-sdk-messaging list-channels --app-instance-arn <arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chime.amazonaws.com --max-results 20
```

## Escalation Checklist

1. Meeting ID and media region
2. Attendee count and connection status
3. Client SDK version and platform
4. Network configuration (UDP/firewall)
5. SIP media application and Lambda function (if PSTN)
6. App instance and user ARNs (if messaging)
7. CloudTrail events for recent API calls
8. Timestamp of when issue started


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Check meeting status and attendees | GREEN |
| Check SIP media applications and phone numbers | GREEN |
| Check app instances and messaging channels | GREEN |
| Review CloudTrail events | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- General troubleshooting may expose data across meetings, messaging, and voice services

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. Rollback steps depend on the specific issue identified during triage — follow the Phase 3 rollback of the matched runbook
2. If multiple changes were made during investigation, revert them in reverse order
3. Document all changes made during troubleshooting for audit purposes
4. Verify rollback by re-running the general diagnostic commands and confirming expected state

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
