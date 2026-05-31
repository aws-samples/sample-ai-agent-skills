# Amazon IVS Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any IVS issue.

## Guardrail 1: IVS Low-Latency and Real-Time Are Different Products
IVS Low-Latency Streaming uses channels, stream keys, and the IVS Player SDK. IVS Real-Time uses stages, participants, and WebRTC. They have different APIs (ivs vs ivs-realtime), different pricing, and different use cases. Do not mix their concepts.

## Guardrail 2: Stream Keys Are Sensitive Credentials
Stream keys authenticate the broadcaster to the IVS ingest endpoint. They must be treated like passwords. If a stream key is compromised, delete it and create a new one. Never log or expose stream keys in client-side code.

## Guardrail 3: IVS Player SDK Required for Low-Latency
Low-latency playback (under 3 seconds) requires the IVS Player SDK. Standard HLS players (HLS.js, Video.js) work but add 10-15 seconds of latency. The IVS Player SDK uses WebRTC for sub-second delivery when available.

## Guardrail 4: IVS Is Available in Limited Regions
IVS is not available in all AWS regions. Check region availability before creating resources. Channels, chat rooms, and stages must be created in supported regions. Recording S3 buckets must be in the same region as the channel.

## Guardrail 5: Recording Configuration Must Be Attached Before Streaming
Auto-recording requires a recording configuration attached to the channel before the stream starts. Attaching a recording config to a channel with an active stream does not record the current stream. The next stream will be recorded.

## Guardrail 6: Chat Tokens Are Short-Lived
IVS Chat tokens are generated server-side and have a configurable expiration (1 minute to 180 minutes). Tokens must be generated per user session. Expired tokens require new token generation. Do not hardcode or cache tokens long-term.

## Guardrail 7: Ingest Settings Affect Quality and Stability
IVS supports specific ingest configurations. Maximum ingest bitrate depends on channel type (BASIC: 1.5 Mbps, STANDARD: 8.5 Mbps, ADVANCED: varies). Exceeding limits causes stream rejection or quality degradation.

## Guardrail 8: Channel Type Determines Features and Limits
BASIC channels have lower bitrate limits and fewer features. STANDARD channels support higher bitrate and recording. ADVANCED channels support additional features like multiple-host. Channel type cannot be changed — must create new channel.

## Guardrail 9: Real-Time Stages Have Participant Limits
IVS Real-Time stages have limits on publishers and subscribers. Default limits vary by stage configuration. Exceeding participant limits causes join failures. Check service quotas for current limits.

## Guardrail 10: Playback URL Is Derived from Channel ARN
The playback URL is returned when creating a channel. It follows a specific format based on the channel. Do not construct playback URLs manually — always use the URL from the API response.

## Guardrail 11: RTMPS Is Required for Ingest
IVS requires RTMPS (RTMP over TLS) for ingest. Plain RTMP is not supported. Encoders must be configured to use rtmps:// protocol. Using rtmp:// will fail to connect.

## Guardrail 12: Stream Health Events Provide Diagnostic Data
IVS provides stream health events via EventBridge and stream session data via get-stream-session. These include ingest metrics, video/audio quality data, and error information. Always check stream session data before diagnosing quality issues.
