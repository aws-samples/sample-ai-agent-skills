# Amazon Rekognition Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Rekognition issue.

## Guardrail 1: Image Size Limits Are Strict
Images passed via S3 can be up to 15MB. Images passed as base64 bytes are limited to 5MB. Exceeding limits returns InvalidImageFormatException. Resize images before processing. JPEG and PNG are the supported formats.

## Guardrail 2: Video Analysis Is Asynchronous
Video analysis uses Start* APIs (StartFaceDetection, StartLabelDetection, etc.) that return a JobId. Results are retrieved with Get* APIs. Use SNS notification topic to know when the job completes. Polling Get* before completion returns incomplete results.

## Guardrail 3: Face Collections Are Region-Specific
Face collections created in one region are not accessible from another. Collection IDs must be unique within a region. IndexFaces adds face vectors; they persist until explicitly deleted. Collections have a default limit of 20 million faces.

## Guardrail 4: Custom Labels Requires Minimum Training Data
Custom Labels projects need minimum training images per label (typically 10+, more for complex use cases). Training datasets must be properly labeled. Training can take 30 minutes to 24 hours depending on dataset size. Insufficient data produces poor models.

## Guardrail 5: Custom Labels Model Must Be Running for Inference
Custom Labels models must be explicitly started (StartProjectVersion) before inference. Models are billed per inference hour while running. Stop models when not in use. Starting a model takes several minutes.

## Guardrail 6: Content Moderation Confidence Thresholds Need Tuning
DetectModerationLabels returns all detections by default. Set MinConfidence parameter to filter low-confidence results. Thresholds should be tuned based on use case — higher for automated actions, lower for human review queues.

## Guardrail 7: Streaming Processors Require Kinesis Integration
Stream processors need Kinesis Video Streams (input) and Kinesis Data Streams (output). Both must be created before the stream processor. IAM role must have permissions for both Kinesis services. Stream processor processes frames in near-real-time.

## Guardrail 8: API Throttling Varies by Operation
Different Rekognition APIs have different TPS (transactions per second) limits. DetectFaces and DetectLabels have higher limits than SearchFacesByImage. Implement exponential backoff for throttling. Request limit increases through AWS Support.

## Guardrail 9: Video Format Requirements
Video analysis supports MP4, MOV, and AVI formats with H.264 codec. Maximum video file size is 10GB. Maximum duration varies by API. Videos must be in S3 (not passed directly). S3 bucket must be in the same region as the API call.

## Guardrail 10: Face Comparison vs Face Search Are Different
CompareFaces compares two specific images. SearchFacesByImage searches a face collection. They serve different use cases. CompareFaces doesn't require a collection. SearchFacesByImage requires a pre-populated collection.

## Guardrail 11: DetectText Has Word Limits
DetectText can detect up to 100 words per image. For document text extraction, use Amazon Textract instead. DetectText is optimized for text in natural scenes (signs, labels), not documents.

## Guardrail 12: PPE Detection Has Specific Requirements
DetectProtectiveEquipment detects face covers, hand covers, and head covers. It requires clear visibility of the person. Confidence thresholds should be set based on safety requirements. Not all PPE types are supported.
