# AWS Elemental MediaPackage Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any MediaPackage issue.

## Guardrail 1: MediaPackage v1 and v2 Are Separate Services
MediaPackage v1 uses `aws mediapackage` CLI commands. MediaPackage v2 uses `aws mediapackagev2` CLI commands. They have different APIs, resource models, and features. Do not mix v1 and v2 commands or assume feature parity.

## Guardrail 2: Channels and Origin Endpoints Are Separate Resources
A channel receives ingest content. Origin endpoints define packaging format and serve content to players. One channel can have multiple origin endpoints (HLS, DASH, CMAF). Players connect to origin endpoint URLs, not channel URLs.

## Guardrail 3: Ingest Authentication Uses WebDAV Credentials
MediaPackage v1 ingest endpoints use username/password authentication over HTTPS. These credentials are generated when the channel is created. MediaLive must be configured with these credentials to push content. Rotating credentials requires updating the source (MediaLive).

## Guardrail 4: CDN Authorization Requires Matching Secrets
CDN authorization uses a shared secret. CloudFront sends a custom header with the secret value. MediaPackage validates the header. Both must use the same secret value stored in Secrets Manager. Mismatched secrets cause 403 errors.

## Guardrail 5: DRM Requires SPEKE Key Provider
MediaPackage DRM uses SPEKE protocol with an external key provider. MediaPackage does not manage DRM keys. The key provider URL must be accessible from MediaPackage. API Gateway + Lambda is a common SPEKE implementation pattern.

## Guardrail 6: Harvest Jobs Require S3 Write Access
Harvest jobs create VOD assets in S3. The MediaPackage IAM role must have s3:PutObject and s3:GetBucketLocation permissions on the destination bucket. Without proper permissions, harvest jobs fail silently or with access denied.

## Guardrail 7: Origin Endpoint Packaging Settings Are Format-Specific
HLS endpoints produce .m3u8 manifests with .ts segments. DASH endpoints produce .mpd manifests with .mp4 segments. CMAF endpoints produce .m3u8 manifests with .mp4 segments. Each format has specific configuration options that don't apply to others.

## Guardrail 8: Startover and Time-Delay Windows Affect Availability
Startover window defines how far back viewers can seek. Time-delay adds latency to the live edge. These are per-origin-endpoint settings. Content outside the startover window is not available for playback or harvest.

## Guardrail 9: CORS Must Be Configured for Browser Playback
Browser-based players require CORS headers on MediaPackage responses. Configure allowed origins on the origin endpoint. Missing CORS causes playback failures in web players but works fine in native apps.

## Guardrail 10: Segment Duration Affects Latency and Compatibility
Shorter segments reduce latency but increase request count. Longer segments increase latency but improve CDN efficiency. Segment duration is configured on the origin endpoint and must be compatible with the source segment duration from MediaLive.

## Guardrail 11: CloudFront Origin Must Point to Origin Endpoint URL
CloudFront origin must use the origin endpoint URL (not the channel URL). The origin path must match the endpoint path. Using the wrong URL results in 404 errors or serving the wrong content format.

## Guardrail 12: Input Redundancy Requires Two Ingest Endpoints
MediaPackage channels have two ingest endpoints (primary and secondary) for redundancy. MediaLive STANDARD channels should send to both endpoints. If only one endpoint receives content, there's no ingest redundancy.
