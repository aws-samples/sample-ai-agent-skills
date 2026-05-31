# AWS Network Manager / Cloud WAN Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Network Manager or Cloud WAN issue.

## Guardrail 1: Core Network Policy Must Be Executed
Creating or updating a core network policy does NOT activate it. The policy generates a change set that must be explicitly executed via `execute-core-network-change-set`. Until executed, the previous policy version remains active. Never assume a newly created policy is in effect.

## Guardrail 2: Attachment Lifecycle States Are Sequential
Attachments follow a strict lifecycle: CREATING → AVAILABLE → UPDATING → DELETING. An attachment in FAILED state cannot be recovered — it must be deleted and recreated. An attachment in REJECTED state was denied by the core network policy's attachment-policies section.

## Guardrail 3: Peering Requires Mutual Acceptance
Peering between two core networks requires both sides to accept the peering request. A peering stuck in CREATING typically means the remote core network owner has not accepted. Cross-account peering also requires RAM resource sharing.

## Guardrail 4: Route Analysis Reflects Executed Policy Only
The `get-network-routes` API returns routes based on the currently EXECUTED policy version. If you have pending policy changes that have not been executed, route analysis will NOT reflect those changes. Always verify which policy version is active.

## Guardrail 5: Transit Gateway Registration Is Regional
A transit gateway must be registered in the global network before it can be used as a core network attachment. Registration is per-region — a TGW in us-east-1 must be registered separately from one in eu-west-1. Registration does not automatically create attachments.

## Guardrail 6: Segments Isolate Traffic by Default
Core network segments provide network isolation. Traffic between segments is blocked unless explicitly allowed via segment-actions (share or route-between) in the core network policy. Do not assume cross-segment connectivity exists without verifying the policy.

## Guardrail 7: VPN Attachments Require Tunnel Configuration
Site-to-site VPN attachments to a core network create VPN tunnels that must be configured on the customer gateway side. The VPN attachment state being AVAILABLE does not mean the tunnel is UP — check tunnel status separately via `aws ec2 describe-vpn-connections`.

## Guardrail 8: Network Function Groups Require Specific Policy Configuration
Network function groups (NFGs) allow service insertion (firewalls, IDS/IPS) in the traffic path. NFGs require specific core network policy configuration including `network-function-groups` and `service-insertion` sections. Misconfigured NFGs silently drop traffic.

## Guardrail 9: Telemetry Data Has Propagation Delay
Network Manager telemetry (bandwidth, latency, packet loss) has a propagation delay of up to 5 minutes. Missing telemetry data does not necessarily indicate a network problem — it may indicate the telemetry source (CloudWatch agent, VPN metrics) is not configured or the data has not propagated yet.

## Guardrail 10: Policy Versions Are Immutable
Each core network policy version is immutable once created. You cannot modify an existing version — you must create a new version. The `list-core-network-policy-versions` API shows all versions. Only one version can be LIVE (executed) at a time.

## Guardrail 11: Global Network Is a Logical Container
A global network is a logical container — it does not create any physical infrastructure. It groups transit gateways, core networks, and other resources for management purposes. Deleting a global network requires removing all registered resources first.

## Guardrail 12: Edge Locations Map to AWS Regions
Core network edge locations correspond to AWS Regions. When specifying edge locations in policies, use the Region code (e.g., us-east-1), not the Region name. Invalid edge locations cause policy validation failures that may not have clear error messages.
