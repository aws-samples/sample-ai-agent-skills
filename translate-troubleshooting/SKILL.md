---
name: translate-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Translate
  problems by analyzing translation errors, custom terminology,
  parallel data, async batch translation, real-time translation,
  language detection, formality settings, profanity masking, and
  following structured runbooks. Activate when: translation errors,
  custom terminology not applied, parallel data training failures,
  batch translation job failures, real-time translation issues,
  language pair not supported, or the user says something is wrong
  with Translate without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Translate, S3, IAM, CloudWatch,
  and CloudTrail permissions.
---

# Amazon Translate Diagnostics

## When to use

Any Amazon Translate investigation where the console alone is insufficient — real-time translation, batch translation, custom terminology, parallel data customization, or language detection.

## Investigation workflow

### Step 1 — Collect and triage

```
aws translate list-terminologies
aws translate list-parallel-data
aws translate list-text-translation-jobs
aws translate translate-text --text "Hello world" --source-language-code en --target-language-code es
```

### Step 2 — Domain deep dive

```
aws translate get-terminology --name <name> --terminology-data-format CSV
aws translate get-parallel-data --name <name>
aws translate describe-text-translation-job --job-id <job-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=translate.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Translate issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `translate translate-text` | Real-time translation |
| `translate list-terminologies` | List custom terminologies |
| `translate list-parallel-data` | List parallel data resources |
| `translate start-text-translation-job` | Start batch translation |
| `translate describe-text-translation-job` | Check batch job status |
| `translate list-languages` | List supported languages |

## Gotchas: Amazon Translate

- Real-time translation has a 10,000 byte limit per request. Use batch for larger documents.
- Custom terminology overrides neural translation for specific terms. Terms are case-sensitive.
- Parallel data trains custom translation models. Training takes time and may not improve all language pairs.
- Batch translation requires S3 input/output. Supports TXT, HTML, DOCX, XLIFF formats.
- Not all language pairs are supported. Check supported pairs before implementation.
- Formality setting only works for specific target languages (de, es, fr, hi, it, ja, nl, pl, pt).
- Auto language detection adds latency. Specify source language when known.
- Profanity masking replaces profane words with a grawlix string (?$#@$).

## Anti-hallucination rules

1. Always cite specific terminology names, job IDs, or API responses as evidence.
2. Real-time has 10KB limit. Never assume unlimited text input.
3. Custom terminology is case-sensitive. Never assume case-insensitive matching.
4. Not all language pairs supported. Never assume universal language support.
5. Formality works for specific languages only. Never assume universal formality support.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Real-Time | A1-A2 | Translation errors, language pairs |
| B — Custom Terminology | B1-B2 | Terminology creation, term application |
| C — Parallel Data | C1-C2 | Parallel data creation, custom models |
| D — Batch Translation | D1-D2 | Batch jobs, format issues |
| E — Features | E1-E2 | Formality, profanity masking |
| Z — Catch-All | Z1 | General troubleshooting |
