---
name: pinpoint-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Pinpoint problems
  by analyzing campaigns, segments, channels (email/SMS/push), analytics,
  journeys, templates, deliverability, event streams, phone number
  management, opt-out handling, and following structured runbooks.
  Activate when: campaign delivery failures, segment creation issues,
  email bounces, SMS delivery failures, push notification errors,
  journey execution problems, template rendering issues, low
  deliverability, event stream failures, phone number provisioning,
  opt-out/consent issues, or the user says something is wrong with
  Pinpoint without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Pinpoint, Pinpoint Email,
  Pinpoint SMS Voice V2, SES, SNS, CloudWatch, CloudTrail, and
  Kinesis permissions. Some operations require verified identities
  and phone number provisioning.
---

# Amazon Pinpoint Diagnostics

## When to use

Any Amazon Pinpoint investigation where the console alone is insufficient — campaign delivery, segment management, channel configuration, journey orchestration, template management, deliverability optimization, or event analytics.

## Investigation workflow

### Step 1 — Collect and triage

```
aws pinpoint get-apps
aws pinpoint get-app --application-id <app-id>
aws pinpoint get-channels --application-id <app-id>
aws pinpoint get-campaigns --application-id <app-id>
```

### Step 2 — Domain deep dive

```
aws pinpoint get-segments --application-id <app-id>
aws pinpoint get-journey --application-id <app-id> --journey-id <journey-id>
aws pinpoint get-email-channel --application-id <app-id>
aws pinpoint get-sms-channel --application-id <app-id>
aws pinpoint-sms-voice-v2 describe-phone-numbers
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=pinpoint.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Pinpoint --metric-name CampaignSendAttempts --dimensions Name=ApplicationId,Value=<app-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws pinpoint get-campaign-activities --application-id <app-id> --campaign-id <campaign-id>
```

Read `references/guardrails.md` before concluding on any Pinpoint issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `pinpoint get-apps` | List Pinpoint applications |
| `pinpoint get-campaigns` | List campaigns and status |
| `pinpoint get-segments` | List segments and sizes |
| `pinpoint get-journey` | Get journey configuration |
| `pinpoint get-email-channel` | Check email channel config |
| `pinpoint get-sms-channel` | Check SMS channel config |
| `pinpoint-sms-voice-v2 describe-phone-numbers` | Check phone numbers |
| `pinpoint get-campaign-activities` | Campaign execution details |
| CloudWatch metrics | Monitor delivery, bounces, complaints |

## Gotchas: Amazon Pinpoint

- Email channel requires verified identity (domain or email) through SES. Without verification, emails are not sent.
- SMS requires phone number provisioning and may need 10DLC registration for US traffic. Unregistered numbers get filtered.
- Campaign scheduling uses UTC. Timezone mismatches cause campaigns to fire at unexpected times.
- Segments are eventually consistent. Dynamic segments may not reflect recent endpoint updates immediately.
- Journey activities have a 6-month maximum duration. Journeys cannot run indefinitely.
- Push notifications require platform credentials (APNs certificate/key, FCM API key) configured per channel.
- Event streams to Kinesis must have proper IAM roles. Missing permissions cause silent event loss.
- SMS spending limits default to $1.00/month. Must request increase for production use.

## Anti-hallucination rules

1. Always cite specific application IDs, campaign IDs, or API responses as evidence.
2. Email requires SES verified identity. Never assume email is ready without verification.
3. SMS requires phone number provisioning and possible 10DLC registration.
4. Campaign metrics are not real-time. Allow time for metrics to populate.
5. Journey execution is asynchronous. Never assume immediate completion.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Campaigns | A1-A3 | Campaign creation, delivery failures, scheduling |
| B — Segments | B1-B2 | Segment creation, dynamic segment issues |
| C — Email | C1-C3 | Email channel, bounces, deliverability |
| D — SMS | D1-D2 | SMS delivery, phone number issues |
| E — Push | E1-E2 | Push notification config, delivery failures |
| F — Journeys | F1-F2 | Journey execution, activity failures |
| G — Templates | G1 | Template rendering issues |
| H — Analytics | H1 | Event stream and analytics |
| Z — Catch-All | Z1 | General troubleshooting |
