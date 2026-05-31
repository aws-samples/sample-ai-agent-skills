# AWS Elemental MediaTailor Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any MediaTailor issue.

## Guardrail 1: SSAI and Channel Assembly Are Separate Features
Server-side ad insertion (SSAI) personalizes ads in live/VOD streams. Channel assembly creates linear channels from VOD sources. They use different API resources and configurations. Do not confuse playback configurations (SSAI) with channels (channel assembly).

## Guardrail 2: ADS Errors Are Not MediaTailor Errors
Ad decision server (ADS) errors originate from the third-party ad server, not MediaTailor. MediaTailor proxies requests to the ADS URL. If the ADS returns invalid VAST/VMAP, empty responses, or timeouts, the root cause is the ADS, not MediaTailor.

## Guardrail 3: CDN Prefix Configuration Affects All Playback URLs
The CDN configuration (AdSegmentUrlPrefix, ContentSegmentUrlPrefix) rewrites segment URLs in the manifest. Incorrect prefixes cause 403/404 errors during playback. Always verify CDN prefixes match the actual CDN distribution domain and path patterns.

## Guardrail 4: Logs Are Not Enabled by Default
MediaTailor does not log to CloudWatch by default. You must explicitly configure log percentages (1-100%) on each playback configuration. A missing log group does not indicate an error — it indicates logging was never configured.

## Guardrail 5: VAST/VMAP Responses Must Be Valid XML
MediaTailor requires valid VAST 2.0/3.0/4.x or VMAP 1.0 XML from the ADS. Malformed XML, missing required elements, or unsupported VAST versions cause ad insertion failures. Test ADS responses independently before blaming MediaTailor.

## Guardrail 6: Prefetch Does Not Guarantee Ad Fill
Prefetch schedules pre-fetch ad decisions to improve fill rates and reduce latency. They do NOT guarantee 100% ad fill. Prefetch windows must align with program schedules. Expired prefetch decisions are discarded.

## Guardrail 7: Source Locations Require Accessible HTTP Endpoints
Source locations must point to HTTP/HTTPS endpoints accessible from MediaTailor. S3 origins require proper bucket policies or CloudFront distributions. Authentication via SigV4 access is supported for S3 origins but must be explicitly configured.

## Guardrail 8: Channel Assembly Requires VOD Sources
Channel assembly channels play VOD sources in a scheduled order. Live sources cannot be used in channel assembly programs. Each program in a channel schedule must reference a valid VOD source from a configured source location.

## Guardrail 9: Manifest Manipulation Is Format-Specific
HLS and DASH manifests have different manipulation rules. HLS uses .m3u8 manifests with #EXT-X tags. DASH uses .mpd manifests with XML Period elements. Ad markers (SCTE-35, DATERANGE, splice_insert) must match the manifest format.

## Guardrail 10: Bumpers and Slates Serve Different Purposes
Bumpers are short clips that play before/after ad breaks (e.g., "We'll be right back"). Slates fill unfilled ad time when the ADS returns fewer ads than the break duration. Both require valid MP4 files accessible via HTTPS.

## Guardrail 11: Configuration Changes Are Not Instant
After updating a playback configuration or channel, changes may take several minutes to propagate. Do not test immediately after making changes. Allow 2-5 minutes for propagation before concluding a change did not work.

## Guardrail 12: MediaTailor Is Regional
MediaTailor resources (configurations, channels, source locations) are regional. Resources in one region are not visible in another. Ensure you are querying the correct region when troubleshooting.
