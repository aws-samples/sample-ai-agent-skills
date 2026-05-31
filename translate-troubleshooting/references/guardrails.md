# Amazon Translate Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Translate issue.

## Guardrail 1: Real-Time Translation Has 10KB Text Limit
TranslateText accepts up to 10,000 bytes of UTF-8 text per request. For larger documents, use batch translation (StartTextTranslationJob) with S3 input. Exceeding the limit returns TextSizeLimitExceededException.

## Guardrail 2: Custom Terminology Is Case-Sensitive
Custom terminology matches are case-sensitive by default. "AWS" won't match "aws" in the source text. Terminology files must use CSV or TMX format. Maximum 256 terms per terminology resource (default).

## Guardrail 3: Not All Language Pairs Are Supported
Amazon Translate supports 75+ languages but not all pairs are available. Some pairs translate through English as a pivot language. Check supported pairs with ListLanguages. Unsupported pairs return UnsupportedLanguagePairException.

## Guardrail 4: Parallel Data Training Takes Time
Parallel data resources train custom translation models. Training is triggered when parallel data is used in a translation job. Training can take hours. Custom models may not improve all language pairs equally.

## Guardrail 5: Batch Translation Requires S3
Batch translation jobs (StartTextTranslationJob) require S3 input and output locations. Supported input formats: TXT, HTML, DOCX, XLIFF. IAM data access role must have S3 read/write permissions.

## Guardrail 6: Formality Only Works for Specific Languages
The Formality setting (FORMAL/INFORMAL) only works for specific target languages: de, es, fr, hi, it, ja, nl, pl, pt. Using formality with unsupported languages is silently ignored.

## Guardrail 7: Auto Language Detection Adds Latency
Setting source language to "auto" triggers automatic detection, adding latency. Specify the source language code when known for faster translation. Auto-detection may be inaccurate for short text.

## Guardrail 8: Profanity Masking Uses Grawlix
When profanity masking is enabled, profane words are replaced with a grawlix string (?$#@$). This is a simple replacement, not contextual. Profanity detection varies by language.

## Guardrail 9: HTML Translation Preserves Tags
When translating HTML content, Translate preserves HTML tags and only translates text content. Malformed HTML may cause translation errors. Use ContentType="text/html" for HTML input.

## Guardrail 10: Terminology and Parallel Data Are Region-Specific
Custom terminologies and parallel data resources are region-specific. They must be created in the same region where translation is performed. They cannot be shared across regions.

## Guardrail 11: Batch Job Input Format Matters
Batch jobs expect one document per file in the input S3 prefix. All files in the prefix are processed. Mixing formats in the same job may cause errors. Specify ContentType correctly.

## Guardrail 12: Translation Quality Varies by Language Pair
Neural machine translation quality varies significantly by language pair. High-resource pairs (en-es, en-fr) generally produce better results than low-resource pairs. Custom parallel data can improve specific domain translations.
