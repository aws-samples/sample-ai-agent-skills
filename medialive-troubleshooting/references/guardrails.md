# AWS Elemental MediaLive Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any MediaLive issue.

## Guardrail 1: Channels Have Two Pipelines by Default
STANDARD class channels have two pipelines (PIPELINE_0, PIPELINE_1) for redundancy. A channel can appear RUNNING while one pipeline is in alert state. Always check both pipeline details in describe-channel output. SINGLE_PIPELINE class has only one pipeline.

## Guardrail 2: Input Security Groups Control PUSH Input Access
PUSH inputs (RTMP_PUSH, RTP_PUSH, UDP_PUSH) require input security groups with whitelisted CIDR ranges. If the source IP is not in the security group, packets are silently dropped with no error. Always verify the source IP is whitelisted.

## Guardrail 3: IAM Role Must Trust medialive.amazonaws.com
MediaLive requires an IAM role with a trust policy for medialive.amazonaws.com. The role needs permissions for S3 (output), MediaPackage, MediaStore, CloudWatch Logs, and SSM Parameter Store depending on the channel configuration.

## Guardrail 4: Channel Class Determines Redundancy
STANDARD class provides two pipelines with automatic failover. SINGLE_PIPELINE class has one pipeline and no failover capability. You cannot change channel class after creation. Do not expect failover from SINGLE_PIPELINE channels.

## Guardrail 5: Output Group Type Determines Destination Requirements
Each output group type has specific destination requirements: HLS needs S3/MediaPackage/MediaStore/HTTP URL, Archive needs S3, UDP needs IP:port, Frame Capture needs S3, MediaPackage needs a MediaPackage channel. Mismatched destinations cause channel start failures.

## Guardrail 6: Schedule Actions Are Immutable After Execution
Schedule actions (input switches, image overlays, SCTE-35 messages) cannot be modified once executed. You can only delete future actions. Creating conflicting actions causes unpredictable behavior. Always verify the schedule before adding actions.

## Guardrail 7: Input Switching Requires Multiple Attached Inputs
Input switching via schedule actions requires multiple inputs attached to the channel. The channel must be configured with input attachments before starting. Dynamic input switching only works between pre-attached inputs.

## Guardrail 8: Encoding Changes Require Channel Stop
Most encoding profile changes (codec, resolution, bitrate) require stopping the channel. Some parameters like volume levels can be changed on a running channel via schedule actions. Do not assume all settings are hot-configurable.

## Guardrail 9: GOP Structure Affects Segment Boundaries
Group of Pictures (GOP) size must align with segment duration for HLS/DASH output. Misaligned GOP causes variable segment durations and playback issues. Segment duration should be a multiple of GOP size in seconds.

## Guardrail 10: PULL Inputs Require Accessible Source URLs
PULL inputs (HLS_PULL, MP4_PULL, RTMP_PULL) require MediaLive to reach the source URL. The source must be accessible from AWS IP ranges. Authentication credentials must be stored in SSM Parameter Store, not inline.

## Guardrail 11: Pipeline Locking Groups Synchronize Outputs
When using STANDARD class with two pipelines, pipeline locking groups ensure both pipelines produce identical output timing. Misconfigured locking groups cause output desynchronization between pipelines.

## Guardrail 12: CloudWatch Metrics Are Per-Pipeline
MediaLive CloudWatch metrics are reported per pipeline (PIPELINE_0, PIPELINE_1). When querying metrics, specify the Pipeline dimension. Aggregate metrics may mask single-pipeline issues.
