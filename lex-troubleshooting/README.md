# Amazon Lex Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Lex problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Lex when the console alone isn't enough — bot creation and building, intent recognition, slot filling, Lambda fulfillment, V2 migration, streaming conversations, channel integrations, and conversation analytics.

### Activate When

- Bot build failures
- Intent not recognized or wrong intent matched
- Slot not filled or wrong value captured
- Lambda fulfillment errors or timeouts
- V1 to V2 migration issues
- Streaming conversation failures
- Channel integration problems (Connect, Slack, etc.)
- Conversation log issues
- Bot alias or versioning errors
- Locale configuration problems
- General Lex errors without clear symptoms

---

## Skill Structure

```
lex-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-bot-creation.md
    ├── A2-build-failures.md
    ├── A3-versioning.md
    ├── B1-intent-matching.md
    ├── B2-missed-utterances.md
    ├── C1-slot-resolution.md
    ├── C2-custom-slot-types.md
    ├── D1-lambda-fulfillment.md
    ├── D2-code-hooks.md
    ├── E1-v2-migration.md
    ├── E2-api-differences.md
    ├── F1-streaming.md
    ├── G1-channel-integrations.md
    ├── G2-connect-integration.md
    ├── H1-conversation-logs.md
    ├── H2-analytics.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Bot Management** | A1–A3 | Bot creation, build failures, versioning |
| **B — Intent Recognition** | B1–B2 | Intent matching, missed utterances |
| **C — Slot Filling** | C1–C2 | Slot resolution, custom slot types |
| **D — Fulfillment** | D1–D2 | Lambda fulfillment, code hooks |
| **E — Migration** | E1–E2 | V1 to V2 migration, API differences |
| **F — Streaming** | F1 | Streaming conversations |
| **G — Channels** | G1–G2 | Channel integrations, Connect integration |
| **H — Logging** | H1–H2 | Conversation logs, analytics |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering bot build requirements, V2 vs V1 API separation, locale-specific slot types, Lambda timeout limits, conversation log IAM, alias versioning, streaming requirements, FallbackIntent behavior, NLU confidence thresholds, session management, bot locale build order, and multi-turn conversation state.

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `lex:*`, `lambda:InvokeFunction`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Bot created and built in target locale

---

## License

MIT-0
