# Amazon Polly Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Polly issue.

## Guardrail 1: Standard vs Neural vs Long-Form vs Generative Engines
Polly has multiple engines: standard (concatenative), neural (deep learning), long-form (optimized for long content), and generative (newest, most natural). Not all voices support all engines. Neural voices sound more natural but have different character limits and pricing.

## Guardrail 2: SSML Must Be Valid XML
SSML input must be well-formed XML wrapped in <speak> tags. Unclosed tags, invalid attributes, or malformed XML cause synthesis failures. SSML tags are case-sensitive. Not all SSML tags are supported — check Polly's supported SSML tag list.

## Guardrail 3: Character Limits Vary by API
SynthesizeSpeech has a 3000 character limit for text input and 6000 for SSML. StartSpeechSynthesisTask (async) supports up to 200,000 characters. Exceeding limits returns a validation error. Character count includes SSML tags for SSML input.

## Guardrail 4: Neural Voices Have Language Restrictions
Neural engine voices are not available for all languages. Some languages only have standard voices. Check describe-voices with engine filter to verify availability. Using a neural voice for an unsupported language returns an error.

## Guardrail 5: Lexicons Use PLS Format
Lexicons use the Pronunciation Lexicon Specification (PLS) XML format, not plain text. Each lexicon has a maximum size. Lexicons are region-specific. Up to 5 lexicons can be applied per synthesis request. Lexicon names must be unique per region.

## Guardrail 6: Async Tasks Require S3 Output
StartSpeechSynthesisTask writes output to an S3 bucket. The Polly service role must have s3:PutObject permission on the bucket. The bucket must exist in the same region. Task status can be polled via GetSpeechSynthesisTask.

## Guardrail 7: Output Formats Have Different Use Cases
Supported formats: mp3, ogg_vorbis, pcm, json (speech marks). Not all formats support all sample rates. PCM is raw audio without headers. JSON output provides speech marks (word, sentence, SSML, viseme timing) not audio.

## Guardrail 8: Speech Marks Are Metadata Not Audio
Speech marks (json output format) provide timing information for words, sentences, SSML marks, and visemes. They are returned as JSON, not audio. Speech marks and audio must be requested in separate API calls.

## Guardrail 9: Rate Limits Apply Per Account
Polly has per-account rate limits for synthesis requests. Throttling returns HTTP 429. Limits differ between SynthesizeSpeech and StartSpeechSynthesisTask. Request limit increases via Service Quotas.

## Guardrail 10: Voice IDs Are Case-Sensitive
Voice IDs (Joanna, Matthew, etc.) are case-sensitive. Using wrong case returns an error. Available voices vary by region. Use describe-voices to get the exact voice ID list for your region.

## Guardrail 11: SSML Prosody Controls Have Ranges
SSML <prosody> tag supports rate, pitch, and volume adjustments. Each has specific allowed ranges. Values outside ranges are clamped or cause errors. Rate: x-slow to x-fast or percentage. Pitch: x-low to x-high or percentage. Volume: silent to x-loud or +/-dB.

## Guardrail 12: Bilingual Voices Have Specific Behavior
Some voices support multiple languages. The language is selected via the language-code parameter or SSML <lang> tag. Mixing languages in a single request requires SSML <lang> tags. Not all voices support bilingual synthesis.
