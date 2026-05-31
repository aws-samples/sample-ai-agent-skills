---
name: lex-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Lex problems
  by analyzing bot creation, intent recognition, slot filling, Lambda
  fulfillment, V2 migration, streaming conversations, channel
  integrations, conversation logs, bot versioning, locale configuration,
  and following structured runbooks. Activate when: bot build failures,
  intent not recognized, slot not filled, Lambda fulfillment errors,
  V1 to V2 migration issues, streaming conversation failures, channel
  integration problems, conversation log issues, bot alias errors,
  locale configuration problems, or the user says something is wrong
  with Lex without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Lex V2, Lex V1 (for migration),
  Lambda, CloudWatch, CloudTrail, IAM, and optionally Connect, Slack,
  or other channel permissions.
---

# Amazon Lex Diagnostics

## When to use

Any Amazon Lex investigation where the console alone is insufficient — bot creation and building, intent recognition, slot filling, Lambda fulfillment, V2 migration, streaming, channel integrations, or conversation analytics.

## Investigation workflow

### Step 1 — Collect and triage

```
aws lexv2-models list-bots
aws lexv2-models describe-bot --bot-id <bot-id>
aws lexv2-models list-bot-aliases --bot-id <bot-id>
aws lexv2-models list-bot-locales --bot-id <bot-id>
```

### Step 2 — Domain deep dive

```
aws lexv2-models list-intents --bot-id <bot-id> --bot-version DRAFT --locale-id en_US
aws lexv2-models list-slots --bot-id <bot-id> --bot-version DRAFT --locale-id en_US --intent-id <intent-id>
aws lexv2-models describe-bot-alias --bot-id <bot-id> --bot-alias-id <alias-id>
aws lexv2-runtime recognize-text --bot-id <bot-id> --bot-alias-id <alias-id> --locale-id en_US --session-id test --text "test utterance"
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=models.lex.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Lex --metric-name MissedUtteranceCount --dimensions Name=BotId,Value=<bot-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws lexv2-models list-bot-versions --bot-id <bot-id>
```

Read `references/guardrails.md` before concluding on any Lex issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `lexv2-models list-bots` | List all bots |
| `lexv2-models describe-bot` | Get bot configuration |
| `lexv2-models list-intents` | List intents in a locale |
| `lexv2-models list-slots` | List slots for an intent |
| `lexv2-runtime recognize-text` | Test bot with text input |
| `lexv2-runtime recognize-utterance` | Test bot with audio/text |
| `lexv2-models build-bot-locale` | Trigger bot build |
| CloudWatch metrics | Monitor recognition, missed utterances |

## Gotchas: Amazon Lex

- Bot must be built after any change to intents, slots, or utterances. Changes are not live until build completes.
- V2 API is completely different from V1. V2 uses bot-id/alias-id, V1 uses bot-name/alias-name. They are not interchangeable.
- Slot types are locale-specific. Built-in slot types vary by locale. Custom slot types must be created per locale.
- Lambda fulfillment timeout is 30 seconds max. Long-running operations must be handled asynchronously.
- Conversation logs require IAM role with CloudWatch Logs or S3 permissions. Missing permissions cause silent log loss.
- Bot aliases point to specific versions. DRAFT is for testing only. Production should use numbered versions.
- Streaming conversations require WebSocket connections. HTTP/2 streaming has different requirements than REST API.
- FallbackIntent catches unrecognized utterances. Without it, unrecognized input returns an error.

## Anti-hallucination rules

1. Always cite specific bot IDs, intent IDs, or API responses as evidence.
2. Bot must be rebuilt after changes. Never assume changes are live without a build.
3. V2 and V1 APIs are completely separate. Never mix V1 and V2 identifiers.
4. Lambda fulfillment has a 30-second timeout. Never assume longer execution.
5. Slot types are locale-specific. Never assume cross-locale compatibility.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Bot Management | A1-A3 | Bot creation, build failures, versioning |
| B — Intent Recognition | B1-B2 | Intent matching, missed utterances |
| C — Slot Filling | C1-C2 | Slot resolution, custom slot types |
| D — Fulfillment | D1-D2 | Lambda fulfillment, code hooks |
| E — Migration | E1-E2 | V1 to V2 migration, API differences |
| F — Streaming | F1 | Streaming conversations |
| G — Channels | G1-G2 | Channel integrations, Connect integration |
| H — Logging | H1-H2 | Conversation logs, analytics |
| Z — Catch-All | Z1 | General troubleshooting |
