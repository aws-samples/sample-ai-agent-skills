---
title: "E2 — Lex Bot Failures"
description: "Diagnose Lex bot integration failures in Connect contact flows"
status: active
severity: HIGH
triggers:
  - "Lex bot not responding"
  - "Bot intent not recognized"
  - "Get customer input Lex error"
  - "Voice bot failure"
owner: devops-agent
objective: "Fix Lex bot integration in contact flows"
context: "Connect uses Lex bots in 'Get customer input' blocks for natural language understanding. The bot must be in the same region as the Connect instance. Voice interactions use streaming. Failures include: bot not found, intent not matched, region mismatch, or permission errors."
---

## Phase 1 — Triage

MUST:
- Verify the Lex bot is associated with the instance: `aws connect list-lex-bots --instance-id <id>`
- Check the Lex bot is in the same region as the Connect instance
- Verify the bot alias is published and active
- Check contact flow logs for Lex-related errors

SHOULD:
- Test the Lex bot directly in the Lex console
- Check if the bot's intents and utterances match expected inputs
- Verify the bot's IAM permissions
- Check if the bot version/alias referenced in the flow is correct

MAY:
- Check Lex bot CloudWatch metrics for errors
- Review the bot's conversation logs
- Test with different utterances to check intent matching

## Phase 2 — Remediate

MUST:
- Deploy the Lex bot in the same region as Connect
- Associate the bot with the Connect instance
- Verify the bot alias in the contact flow matches the published alias

SHOULD:
- Test the bot with voice input (not just text)
- Add fallback intents for unrecognized utterances
- Configure the "Get customer input" block's error branch

MAY:
- Implement Lex bot versioning and testing strategy
- Add conversation logging for debugging
- Implement DTMF fallback for Lex failures

## Common Issues

- symptoms: "Lex bot works in console but not in Connect"
  diagnosis: "Bot is in a different region than the Connect instance."
  resolution: "Deploy the bot in the same region as the Connect instance."

- symptoms: "Bot doesn't understand caller's speech"
  diagnosis: "Utterances don't match caller's phrasing, or audio quality is poor."
  resolution: "Add more utterance variations. Check audio quality. Consider adding DTMF fallback."

- symptoms: "'Get customer input' block goes to Error branch"
  diagnosis: "Bot alias not found, permission error, or bot not associated with instance."
  resolution: "Verify bot association, alias, and permissions."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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

1. If a contact flow was modified, publish the previous saved version of the flow from the Connect console
2. If Lambda or Lex integration was changed, restore the original function/bot association in the contact flow
3. If agent routing or security profiles were modified, restore original profile assignments
4. If authentication or permission settings were changed, revert identity provider configuration
## Output Format

```yaml
root_cause: "lex_bot — <specific cause>"
evidence:
  - type: contact_flow_logs
    content: "<Lex error details>"
severity: HIGH
mitigation:
  immediate: "Fix bot configuration or region"
  long_term: "Implement bot testing and monitoring"
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
