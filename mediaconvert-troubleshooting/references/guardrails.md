# AWS Elemental MediaConvert Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any MediaConvert issue.

## Guardrail 1: Always Use the Account-Specific Endpoint
MediaConvert requires an account-specific endpoint URL for all API calls except describe-endpoints. The endpoint format is https://<account-hash>.mediaconvert.<region>.amazonaws.com. Using the generic endpoint will fail. Always call describe-endpoints first.

## Guardrail 2: Job Error Messages Contain the Specific Failure Reason
The get-job response includes errorCode and errorMessage fields with the specific failure reason. Always check these before diagnosing. Common error codes: 1000 (input error), 1020 (codec not supported), 1040 (S3 access), 1060 (output error). Do not guess the failure reason.

## Guardrail 3: IAM Role Must Trust mediaconvert.amazonaws.com
MediaConvert requires an IAM role with a trust policy allowing mediaconvert.amazonaws.com to assume it. The role needs s3:GetObject for input buckets and s3:PutObject for output buckets. Without the correct trust policy, jobs fail immediately.

## Guardrail 4: S3 Bucket Access Requires Both IAM and Bucket Policy
The MediaConvert role needs IAM permissions for S3, AND the S3 bucket policy must not deny access. For cross-account scenarios, both the IAM role policy and the bucket policy must explicitly allow access. Check both when diagnosing S3 access errors.

## Guardrail 5: Not All Video Codecs and Containers Are Supported
MediaConvert supports specific input and output codecs. Common supported inputs: H.264, H.265, MPEG-2, ProRes, AV1. Common containers: MP4, MKV, MOV, MXF, TS. Not all codec/container combinations are valid. Always verify format support before diagnosing codec errors.

## Guardrail 6: Output Groups Define the Delivery Format
Each job can have multiple output groups: File Group (single files), Apple HLS, DASH ISO, CMAF, MS Smooth Streaming. Each group type has specific requirements and limitations. HLS requires H.264 or H.265 video codec. DASH supports additional codecs. Do not mix incompatible codecs with output group types.

## Guardrail 7: On-Demand vs Reserved Queues Have Different Behavior
On-demand queues process jobs with no upfront commitment and scale automatically. Reserved queues have a fixed number of Reserved Transcode Slots (RTS) with committed pricing. Reserved queues do NOT auto-scale beyond the RTS count. Do not expect reserved queues to handle burst traffic beyond their RTS capacity.

## Guardrail 8: Accelerated Transcoding Has Eligibility Requirements
Accelerated transcoding (professional tier) is faster but costs more. It requires specific input codecs (H.264, H.265, MPEG-2, ProRes), minimum resolution, and specific output codecs. Not all jobs qualify. If a job doesn't qualify, it falls back to standard transcoding or fails.

## Guardrail 9: DRM Requires a SPEKE-Compatible Key Provider
MediaConvert DRM uses SPEKE (Secure Packager and Encoder Key Exchange) protocol. You must have a SPEKE-compatible key provider (e.g., AWS SPEKE Reference Server, PallyCon, BuyDRM). MediaConvert does not manage DRM keys directly. The key provider URL must be accessible from MediaConvert.

## Guardrail 10: Job Priority Only Applies Within a Queue
Job priority (0-50) determines processing order within a single queue. Higher priority jobs are processed first. Priority does NOT affect processing speed or resource allocation. Jobs in different queues are independent. Default priority is 0.

## Guardrail 11: Audio Track Selection Requires Explicit Configuration
MediaConvert does not automatically select the "best" audio track. If the input has multiple audio tracks, you must explicitly configure which tracks to include in the output using audio selectors. Omitting audio selectors may result in missing or wrong audio in the output.

## Guardrail 12: Resolution and Bitrate Must Be Balanced
Setting high resolution with low bitrate produces poor quality. Setting low resolution with high bitrate wastes bandwidth. MediaConvert does not automatically optimize this relationship. Use QVBR (Quality-Defined Variable Bitrate) rate control for automatic quality optimization, or follow encoding best practices for CBR/VBR.
