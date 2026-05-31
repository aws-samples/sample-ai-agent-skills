# Amazon Transcribe Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Transcribe issue.

## Guardrail 1: Audio Format Must Match MediaFormat Parameter
The MediaFormat parameter must match the actual audio format. Supported: wav, mp3, mp4, flac, ogg, amr, webm. Mismatched format causes job failure. Transcribe does not auto-detect format.

## Guardrail 2: Custom Vocabulary Must Be in READY State
Custom vocabularies take time to process after creation. Status progresses from PENDING to READY. Using a vocabulary before READY causes errors. Check status with GetVocabulary.

## Guardrail 3: Streaming Uses WebSocket or HTTP/2
Streaming transcription requires WebSocket or HTTP/2 connection. Different from batch API. Has specific audio encoding requirements (PCM 16-bit). Connection has idle timeout.

## Guardrail 4: Medical Transcription Is a Separate API
StartMedicalTranscriptionJob is separate from StartTranscriptionJob. Medical API has HIPAA compliance. Don't use standard API for medical content. Medical supports specific specialties.

## Guardrail 5: Call Analytics Requires Stereo Audio
Call Analytics requires stereo audio with separate channels per speaker. Mono audio is not supported. Channel identification maps speakers to channels. Configure ChannelDefinitions correctly.

## Guardrail 6: Content Redaction Has Language Limitations
Content redaction (PII removal) is available for specific languages only (primarily English). Redaction replaces PII with [PII] tags. Not all PII types detected in all languages.

## Guardrail 7: Maximum Audio Duration Is 4 Hours
Standard and medical transcription support up to 4 hours of audio. Longer recordings must be split. Streaming has no fixed duration limit but has connection timeout.

## Guardrail 8: Sample Rate Affects Accuracy
Audio sample rate affects transcription accuracy. 16kHz or higher recommended for telephony. 44.1kHz for broadcast quality. Low sample rates produce lower accuracy.

## Guardrail 9: Speaker Diarization Has Limits
Speaker identification (diarization) supports 2-10 speakers. Setting MaxSpeakerLabels too high or too low affects accuracy. Not available with channel identification simultaneously.

## Guardrail 10: Job Names Must Be Unique
Transcription job names must be unique within an account and region. Reusing a name returns ConflictException. Include timestamps or UUIDs in job names.

## Guardrail 11: Output Is JSON by Default
Transcription output is JSON format by default. Subtitles (SRT, VTT) require explicit configuration. Output location can be default S3 bucket or custom S3 location.

## Guardrail 12: Language Identification Adds Processing Time
Automatic language identification adds processing time. Specify LanguageCode when known. IdentifyLanguage and IdentifyMultipleLanguages have different behaviors.
