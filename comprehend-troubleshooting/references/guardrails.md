# Amazon Comprehend Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Comprehend issue.

## Guardrail 1: Real-Time APIs Have Text Size Limits
Synchronous APIs (DetectEntities, DetectSentiment, etc.) accept up to 100KB of UTF-8 text per request. For larger volumes, use async batch APIs (StartEntitiesDetectionJob, etc.) with S3 input.

## Guardrail 2: Custom Models Require Endpoints for Inference
Custom classifiers and entity recognizers cannot be invoked directly. You must create an endpoint (CreateEndpoint) pointing to the model. Endpoints are billed per inference unit while active. Delete endpoints when not needed.

## Guardrail 3: Custom Classifier Training Data Has Minimum Requirements
Multi-class mode requires minimum 10 documents per class. Multi-label mode requires minimum 10 documents per unique label combination. CSV format: one document per line with label. Augmented manifest format also supported.

## Guardrail 4: Custom Entity Recognizer Needs Annotated Data
Entity recognizers require annotated training data identifying entity locations in text. Annotations format requires entity list and documents. Plain text format requires entity list with type and text. Minimum varies by entity type count.

## Guardrail 5: PII Detection Language Support Varies
PII detection supports English primarily for all entity types. Other languages have limited PII type support. Check language-specific PII type availability before relying on detection. DetectPiiEntities and ContainsPiiEntities have different capabilities.

## Guardrail 6: Async Jobs Require S3 Input/Output
Batch processing jobs (Start*Job) require S3 input and output locations. IAM data access role must have S3 read/write permissions. Input format varies by job type (one document per line or one per file).

## Guardrail 7: Language Code Is Required for Most APIs
Most Comprehend APIs require a language code parameter. Use DetectDominantLanguage first if language is unknown. Supported languages vary by API. Using wrong language code produces poor results.

## Guardrail 8: Topic Modeling Requires Minimum Documents
StartTopicsDetectionJob requires minimum documents for meaningful results (recommended 1000+). Input must be in S3. Each document should be substantial text. Very short documents produce poor topic models.

## Guardrail 9: Sentiment Scores Are Probabilities
DetectSentiment returns scores for POSITIVE, NEGATIVE, NEUTRAL, and MIXED. Scores are probabilities summing to ~1.0. The dominant sentiment has the highest score. Low dominant score indicates ambiguous sentiment.

## Guardrail 10: Endpoints Have Throughput Limits
Custom model endpoints have throughput based on inference units. One inference unit provides baseline throughput. Scale up inference units for higher throughput. Monitor endpoint utilization.

## Guardrail 11: Flywheel Manages Model Lifecycle
Comprehend Flywheel automates model retraining and evaluation. Flywheels manage datasets, training, and model selection. Not all custom models use flywheels — they're optional but recommended for production.

## Guardrail 12: VPC Configuration Is Optional but Affects Access
Comprehend jobs can run in a VPC for data isolation. VPC configuration requires subnets and security groups. VPC jobs need NAT gateway or VPC endpoints for S3 access. Non-VPC jobs access S3 directly.
