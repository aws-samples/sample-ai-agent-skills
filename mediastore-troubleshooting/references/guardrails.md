# AWS Elemental MediaStore Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any MediaStore issue.

## Guardrail 1: MediaStore Is Not S3
MediaStore containers are NOT S3 buckets. They use different APIs (mediastore/mediastore-data), different endpoints, different policy formats, and have different capabilities. Never use S3 CLI commands or S3 policy syntax for MediaStore.

## Guardrail 2: Two API Namespaces — Control Plane and Data Plane
MediaStore has two API namespaces: `mediastore` (control plane for container management, policies, logging) and `mediastore-data` (data plane for object put/get/delete). They use different endpoints. The data plane endpoint is container-specific.

## Guardrail 3: Container Policies Are Not S3 Bucket Policies
MediaStore container policies have a simpler format than S3 bucket policies. They support Action, Principal, Condition, and Resource but with MediaStore-specific actions. Do not copy S3 bucket policy syntax directly.

## Guardrail 4: MediaStore Is Optimized for Live Video
MediaStore is designed for live video origination with consistent low-latency reads. It is NOT a general-purpose storage service. Throughput limits are lower than S3. Use S3 for non-media bulk storage.

## Guardrail 5: Object Paths Must Start with Forward Slash
All object paths in MediaStore must start with `/`. The path structure is flat with folder-like prefixes. There are no nested containers or buckets within a container.

## Guardrail 6: CORS Is Container-Level Only
CORS policies are set at the container level and apply to all objects in the container. There is no per-object or per-path CORS configuration. CORS changes take effect immediately.

## Guardrail 7: Lifecycle Policies Delete Objects Permanently
Lifecycle policies in MediaStore permanently delete objects after the specified retention period. There is no versioning or soft-delete. Deleted objects cannot be recovered. Test lifecycle policies carefully.

## Guardrail 8: Container Deletion Requires Empty Container
MediaStore containers cannot be deleted if they contain objects. All objects must be deleted first. There is no force-delete option. Use lifecycle policies or batch delete for cleanup.

## Guardrail 9: Access Logging Goes to CloudWatch
MediaStore access logging sends logs to CloudWatch Logs, not to S3 (unlike S3 access logging). The log group is `/aws/mediastore/<container-name>`. Logging must be explicitly enabled per container.

## Guardrail 10: Metric Policies Control CloudWatch Metrics Granularity
Metric policies determine which CloudWatch metrics are published and at what granularity (container-level or path-level). Without a metric policy, only basic container-level metrics are available.

## Guardrail 11: CDN Integration Requires Specific Origin Configuration
When using CloudFront with MediaStore, the origin must be configured with the MediaStore container endpoint (not an S3 endpoint). Origin access identity (OAI) is not used — container policies control access.

## Guardrail 12: Throttling Limits Are Per-Container
MediaStore throttling limits (transactions per second) are per-container, not per-account. High-throughput workloads may need multiple containers. Request limit increases through AWS Support for specific containers.
