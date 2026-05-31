---
name: mediapackage-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elemental MediaPackage
  problems by analyzing channel configuration, origin endpoints, HLS/DASH/CMAF
  packaging, DRM, CDN authorization, live-to-VOD, and harvest jobs using
  structured runbooks.
  Activate when: channel creation failures, origin endpoint errors, HLS/DASH/CMAF
  packaging issues, DRM configuration problems, CDN auth failures, live-to-VOD
  issues, harvest job failures, or the user says something is wrong with
  MediaPackage without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with MediaPackage, MediaPackage v2,
  CloudFront, S3, IAM, CloudWatch, CloudTrail permissions.
---

# MediaPackage Diagnostics

## When to use

Any AWS Elemental MediaPackage investigation — channel configuration, origin endpoints, packaging formats, DRM, CDN authorization, live-to-VOD workflows, or harvest jobs.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mediapackage list-channels --max-results 20
aws mediapackage describe-channel --id <channel-id>
aws mediapackage list-origin-endpoints --channel-id <channel-id>
aws cloudwatch get-metric-statistics --namespace AWS/MediaPackage --metric-name IngressBytes --dimensions Name=Channel,Value=<channel-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws mediapackage describe-origin-endpoint --id <endpoint-id>
aws mediapackage list-harvest-jobs --include-channel-id <channel-id>
aws cloudwatch get-metric-statistics --namespace AWS/MediaPackage --metric-name EgressRequestCount --dimensions Name=OriginEndpoint,Value=<endpoint-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 3 — Detailed investigation

```
aws cloudfront get-distribution --id <dist-id> | jq '.Distribution.DistributionConfig.Origins'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mediapackage.amazonaws.com --max-results 10
aws mediapackagev2 list-channels --channel-group-name <group>
```

Read `references/guardrails.md` before concluding on any MediaPackage issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-channel` | Channel config and ingest endpoints |
| `list-origin-endpoints` | All endpoints for a channel |
| `describe-origin-endpoint` | Endpoint packaging config |
| `list-harvest-jobs` | Live-to-VOD harvest status |
| `describe-harvest-job` | Harvest job details |
| `rotate-ingest-endpoint-credentials` | Rotate ingest auth |

## Gotchas: MediaPackage

- MediaPackage v1 and v2 are separate services with different APIs.
- Ingest endpoints use WebDAV with username/password authentication.
- Origin endpoints define the packaging format (HLS, DASH, CMAF, MSS).
- CDN authorization uses a shared secret between CloudFront and MediaPackage.
- Harvest jobs create VOD assets from live content windows.
- Each origin endpoint has its own URL — players connect to origin endpoints, not channels.
- CORS headers must be configured on origin endpoints for browser playback.

## Anti-hallucination rules

1. MediaPackage v1 and v2 have different APIs — don't mix them.
2. Origin endpoints are per-format — one for HLS, one for DASH, etc.
3. CDN authorization requires both CloudFront custom header and MediaPackage secret.
4. Harvest jobs require S3 bucket access — check IAM role permissions.
5. Ingest credentials are per-endpoint — rotating one doesn't affect others.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Channel | A1-A2 | Channel creation, ingest config |
| B — Origin | B1-B2 | Origin endpoints, endpoint errors |
| C — Packaging | C1-C3 | HLS, DASH, CMAF packaging |
| D — DRM | D1-D2 | DRM config, key rotation |
| E — CDN | E1 | CDN authorization |
| F — VOD | F1-F2 | Live-to-VOD, harvest jobs |
| G — v2 | G1 | MediaPackage v2 migration |
| Z — Catch-All | Z1 | General troubleshooting |
