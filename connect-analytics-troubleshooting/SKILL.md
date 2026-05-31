---
name: connect-analytics-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Connect
  Contact Lens and Wisdom problems by analyzing real-time analytics,
  sentiment analysis, transcription, category rules, custom
  vocabulary, knowledge bases, search, evaluation forms, screen
  recording, data lake export, S3 output, third-party integrations,
  and following structured runbooks.
  Activate when: Contact Lens enablement issues, real-time analytics
  errors, sentiment analysis problems, transcript failures, category
  rule errors, custom vocabulary issues, Wisdom knowledge base
  problems, search failures, evaluation form issues, screen recording
  errors, data export problems, or the user says something is wrong
  with Contact Lens or Wisdom.
compatibility: >
  Requires AWS CLI or SDK access with connect:*, wisdom:*,
  s3:Get*, s3:Put*, kinesis:*, cloudwatch:*, iam:Get*, and
  logs:* permissions. Amazon Connect instance admin access may
  be required for Contact Lens and Wisdom configuration.
---

# Amazon Connect Contact Lens & Wisdom Diagnostics

## When to use

Any Amazon Connect Contact Lens or Wisdom investigation — Contact Lens enablement, real-time analytics, sentiment analysis, transcription, category rules, custom vocabulary, Wisdom knowledge bases, search, evaluation forms, screen recording, data lake export, or S3 output configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws connect list-instances --query 'InstanceSummaryList[*].{Id:Id,Arn:Arn,InstanceAlias:InstanceAlias,InstanceStatus:InstanceStatus}'
aws connect list-contact-flow-modules --instance-id <instance-id>
aws connect describe-instance --instance-id <instance-id> --query '{InstanceId:Instance.Id,ContactLensEnabled:Instance.StatusReason}'
```

### Step 2 — Domain deep dive

```
aws connect describe-contact --instance-id <instance-id> --contact-id <contact-id>
aws connect list-realtime-contact-analysis-segments-v2 --instance-id <instance-id> --contact-id <contact-id> --output-type Raw --segment-types Transcript Categories
aws wisdom list-knowledge-bases
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=connect.amazonaws.com --max-results 20
aws connect list-rules --instance-id <instance-id> --publish-status PUBLISHED
aws s3 ls s3://<connect-bucket>/connect/<instance-alias>/ContactLens/
```

Read `references/guardrails.md` before concluding on any Contact Lens or Wisdom issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `connect describe-instance` | Get instance details and feature status |
| `connect describe-contact` | Get contact details including analytics |
| `connect list-realtime-contact-analysis-segments-v2` | Get real-time transcript and categories |
| `connect list-rules` | List Contact Lens rules |
| `wisdom list-knowledge-bases` | List Wisdom knowledge bases |
| `wisdom search-content` | Search Wisdom knowledge base content |
| `connect list-evaluation-forms` | List evaluation forms |

## Gotchas: Amazon Connect Contact Lens & Wisdom

- Contact Lens must be enabled at the instance level AND in the contact flow. Both are required — enabling one without the other produces no analytics.
- Real-time analytics are available ONLY during active calls. Post-contact analytics are available after the call ends and processing completes (typically 5-15 minutes).
- Sentiment analysis provides per-turn and overall sentiment scores. Scores range from -5 (most negative) to +5 (most positive). Neutral is around 0.
- Transcription accuracy depends on audio quality, language, accents, and background noise. Custom vocabulary can improve accuracy for domain-specific terms.
- Category rules use pattern matching on transcripts. Rules are evaluated post-contact unless configured for real-time. Real-time rules have latency.
- Wisdom knowledge bases ingest content from S3, ServiceNow, Salesforce, or other connectors. Content must be indexed before it is searchable.
- Screen recording requires agent desktop configuration and storage in S3. It is separate from call recording and has its own storage and retention settings.

## Anti-hallucination rules

1. Always cite specific instance IDs, contact IDs, or API responses as evidence.
2. Contact Lens requires BOTH instance-level enablement AND contact flow configuration. Never suggest one alone is sufficient.
3. Real-time analytics are only available during active calls. Never suggest accessing real-time data for completed contacts.
4. Wisdom and Contact Lens are separate features. Never conflate knowledge base search with sentiment analysis.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Enablement | A1–A2 | Contact Lens enablement, real-time analytics errors |
| B — Analysis | B1–B2 | Sentiment analysis issues, transcript generation failures |
| C — Rules | C1–C2 | Category rule errors, custom vocabulary issues |
| D — Wisdom | D1–D2 | Knowledge base errors, search failures |
| E — Quality | E1–E2 | Evaluation form issues, screen recording errors |
| F — Export | F1–F2 | Data lake export, S3 output configuration |
| G — Integration | G1 | Third-party integration |
| Z — Catch-All | Z1 | General troubleshooting |
