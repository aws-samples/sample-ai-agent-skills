---
name: polly-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Polly problems
  by analyzing speech synthesis, SSML markup, neural voices, lexicons,
  async tasks, and streaming issues using structured runbooks. Activate
  when: synthesis failures, SSML errors, neural voice issues, lexicon
  problems, async task failures, streaming errors, or the user says
  something is wrong with Polly.
compatibility: >
  Requires AWS CLI or SDK access with Polly, IAM, S3, and CloudWatch
  permissions as applicable.
---

# Amazon Polly Diagnostics

## When to use

Any Amazon Polly investigation — speech synthesis, SSML markup, neural voices, lexicons, async tasks, or streaming troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws polly describe-voices --language-code en-US
aws polly list-lexicons
aws polly list-speech-synthesis-tasks --status inProgress
```

### Step 2 — Domain deep dive

```
aws polly get-lexicon --name <lexicon-name>
aws polly get-speech-synthesis-task --task-id <task-id>
aws polly synthesize-speech --text "test" --output-format mp3 --voice-id Joanna output.mp3
```

### Step 3 — Detailed investigation

```
aws polly synthesize-speech --text-type ssml --text '<speak>test</speak>' --output-format mp3 --voice-id Joanna output.mp3
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=polly.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Polly --metric-name RequestCharacters --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any Polly issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-voices` | List available voices and engines |
| `synthesize-speech` | Test speech synthesis |
| `get-lexicon` | Check lexicon content |
| `get-speech-synthesis-task` | Check async task status |
| `list-speech-synthesis-tasks` | List pending/completed tasks |
| CloudWatch Metrics | Check request rates and errors |

## Anti-hallucination rules

1. Always cite specific voice IDs, task IDs, or API responses as evidence.
2. Neural voices are NOT available for all languages. Check availability first.
3. SSML tags must be valid XML. Never assume HTML-like tolerance.
4. Lexicons use PLS format, not plain text. Never confuse formats.
5. Async synthesis requires S3 output. Never assume local file output.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Synthesis | A1-A2 | Synthesis failures, output format |
| B — SSML | B1-B2 | SSML errors, tag issues |
| C — Voices | C1-C2 | Neural voices, voice selection |
| D — Lexicons | D1-D2 | Lexicon errors, pronunciation |
| E — Async | E1-E2 | Task failures, S3 output |
| Z — Catch-All | Z1-Z2 | General troubleshooting, streaming |
