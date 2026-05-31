# Amazon Polly Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Polly problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Polly — speech synthesis, SSML markup, neural voices, lexicons, async tasks, and streaming.

### Activate When

- Speech synthesis failures or errors
- SSML markup validation errors
- Neural voice unavailability or quality issues
- Lexicon creation or application problems
- Async synthesis task failures
- Streaming synthesis errors
- Output format or encoding issues
- General Polly errors

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Synthesis** | A1–A2 | Synthesis failures, output format |
| **B — SSML** | B1–B2 | SSML errors, tag issues |
| **C — Voices** | C1–C2 | Neural voices, voice selection |
| **D — Lexicons** | D1–D2 | Lexicon errors, pronunciation |
| **E — Async** | E1–E2 | Task failures, S3 output |
| **Z — Catch-All** | Z1–Z2 | General troubleshooting, streaming |

---

## Prerequisites

- AWS CLI v2 with Polly permissions
- Permissions: `polly:SynthesizeSpeech`, `polly:DescribeVoices`, `polly:ListLexicons`, `polly:GetSpeechSynthesisTask`

---

## License

MIT-0
