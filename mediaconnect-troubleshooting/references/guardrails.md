# AWS Elemental MediaConnect Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any MediaConnect issue.

## Guardrail 1: Flows Are Regional and AZ-Specific
MediaConnect flows exist in a single AWS region and availability zone. A flow cannot span multiple regions or AZs. Multi-region delivery requires separate flows connected via entitlements or MediaConnect Gateway bridges. Do not assume a flow is accessible from another region.

## Guardrail 2: Source Failover Requires Two Sources in Different AZs
Source failover requires exactly two sources configured in different availability zones delivering identical content. Both sources must use the same protocol and encoding settings. Single-source flows cannot fail over. Do not suggest failover without verifying dual-source configuration.

## Guardrail 3: Entitlements Share Between Accounts Not Within
Entitlements grant another AWS account permission to use a flow's content as a source. The subscriber account creates their own flow referencing the entitlement ARN. Entitlements do NOT share content within the same account — use outputs for same-account delivery.

## Guardrail 4: VPC Interfaces Require ENI and Security Group Configuration
VPC-based flows use Elastic Network Interfaces (ENIs) in your VPC. Security groups must explicitly allow inbound traffic on the protocol port from the source IP range. ENI limits per subnet and account quotas apply. Do not assume default security groups allow MediaConnect traffic.

## Guardrail 5: Three Encryption Models with Different Requirements
MediaConnect supports three encryption models: SPEKE (requires DRM key provider URL and IAM role), static-key (requires Secrets Manager secret with base64-encoded key), and SRT passphrase (configured on the SRT protocol settings). Each has different setup requirements and failure modes.

## Guardrail 6: Transport Stream Errors Usually Indicate Source Problems
MPEG-TS transport stream errors (continuity counter errors, PID errors, sync byte loss) typically originate from the upstream encoder or source, not from MediaConnect itself. Investigate the source encoder and network path before blaming MediaConnect.

## Guardrail 7: Bandwidth Limits Apply at Flow and Output Level
Each flow has a maximum bitrate setting. Individual outputs also have bandwidth constraints. Exceeding these limits causes packet loss or output failures. The total output bandwidth cannot exceed the flow's source bitrate. Check both flow-level and output-level limits.

## Guardrail 8: Flow Status Transitions Are Not Instant
Starting or stopping a flow takes time. Flow status transitions through STARTING → ACTIVE or STOPPING → STANDBY. Do not assume immediate state changes. Repeated start/stop commands during transitions can cause errors.

## Guardrail 9: Protocols Have Different Network Requirements
Zixi, SRT, RIST, and RTP/FEC each have different port, firewall, and NAT traversal requirements. SRT supports listener and caller modes for NAT traversal. Zixi requires specific port ranges. Do not apply one protocol's network requirements to another.

## Guardrail 10: CloudWatch Metrics Have Minimum 1-Minute Granularity
MediaConnect publishes metrics to CloudWatch at 1-minute intervals. Metrics include SourceBitRate, OutputBitRate, TransportStreamError, and others. Sub-minute issues may not appear in metrics. Use flow logs for detailed packet-level analysis.

## Guardrail 11: Maintenance Windows Affect Flow Availability
AWS schedules maintenance events for MediaConnect infrastructure. During maintenance, flows may experience brief interruptions. Use source failover with dual-AZ sources to maintain availability during maintenance. Check AWS Health Dashboard for scheduled events.

## Guardrail 12: MediaConnect Gateway Is Separate from Standard Flows
MediaConnect Gateway enables on-premises to cloud connectivity using gateway bridges and instances. Gateway configuration is separate from standard flow management. Do not confuse gateway bridges with standard flow entitlements or outputs.
