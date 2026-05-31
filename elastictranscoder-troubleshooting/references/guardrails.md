# Elastic Transcoder Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Elastic Transcoder issue.

## Guardrail 1: Elastic Transcoder Is a Legacy Service
Elastic Transcoder is a legacy service. AWS recommends AWS Elemental MediaConvert for new workloads. However, Elastic Transcoder is still fully supported. Do not suggest migrating to MediaConvert unless the user explicitly asks about migration or the issue cannot be resolved within Elastic Transcoder.

## Guardrail 2: Pipelines Are Regional
Each pipeline is tied to a specific AWS region. Input and output S3 buckets must be in the same region as the pipeline. Cross-region bucket references cause pipeline creation failures or job errors. Always verify region alignment.

## Guardrail 3: Service Role Requires Both Read and Write
The Elastic Transcoder service role needs s3:GetObject on the input bucket AND s3:PutObject on the output bucket. It also needs s3:PutObject for the thumbnail bucket if thumbnails are configured. Missing any permission causes AccessDenied errors on jobs.

## Guardrail 4: System Presets Are Immutable
System presets (IDs starting with "1351620000001-") cannot be modified or deleted. To customize encoding settings, create a new custom preset. Do not suggest editing system presets — it is not possible.

## Guardrail 5: Job Status Progressing Does Not Mean Failure
Jobs in "Progressing" status are actively being processed. Large files or complex encoding can take significant time. Only investigate stuck jobs if they have been in Progressing state for longer than expected based on file size and complexity.

## Guardrail 6: Input File Must Exist Before Job Submission
The input file must exist in the S3 input bucket at the time the job is created. Elastic Transcoder does not wait for files to appear. Jobs submitted before the file upload completes will fail with a 404 error.

## Guardrail 7: Watermarks Require PNG with Alpha Channel
Watermark images must be PNG format with alpha channel transparency. JPEG or other formats cause job failures or are silently ignored. Verify watermark file format before troubleshooting watermark rendering issues.

## Guardrail 8: SNS Notifications Require Topic Permissions
Pipeline SNS notifications require the Elastic Transcoder service role to have sns:Publish permission on the configured SNS topics. The SNS topic policy must also allow the service to publish. Both sides must be configured.

## Guardrail 9: Output Key Prefix Determines File Location
The output key prefix in the pipeline configuration determines where output files are stored in the output bucket. If outputs are missing, check the key prefix configuration — files may be in an unexpected S3 path.

## Guardrail 10: Concurrent Job Limits Exist Per Pipeline
Each pipeline has a limit on concurrent jobs. Exceeding this limit causes jobs to queue, not fail. If jobs appear delayed, check if the pipeline is at capacity before investigating other causes.

## Guardrail 11: Custom Presets Must Match Container Format
Custom preset audio and video codec settings must be compatible with the specified container format. For example, VP8 video codec requires WebM container. Mismatched codec-container combinations cause preset creation failures.

## Guardrail 12: Thumbnails Are Per-Output Configuration
Thumbnail generation is configured per output, not per job. Each output in a job can have different thumbnail settings (interval, format, size). Missing thumbnails may be due to output-level configuration, not job-level settings.
