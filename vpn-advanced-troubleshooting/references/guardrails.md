# AWS Site-to-Site VPN Advanced Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any advanced VPN issue.

## Guardrail 1: Accelerated VPN Requires Transit Gateway
Accelerated VPN uses AWS Global Accelerator to route traffic through the AWS global network. It is ONLY supported with transit gateway-based VPN connections. Virtual private gateway (VGW) VPN connections do NOT support acceleration. Do not suggest enabling acceleration on VGW-based VPNs.

## Guardrail 2: ECMP Requires Multiple VPN Connections
Equal-Cost Multi-Path (ECMP) routing distributes traffic across multiple VPN tunnels. ECMP is ONLY supported with transit gateway VPN attachments and requires multiple VPN connections (not just multiple tunnels within one connection). A single VPN connection with two tunnels does NOT provide ECMP.

## Guardrail 3: Certificate-Based Auth Uses ACM Private CA
Certificate-based VPN authentication uses certificates from AWS Certificate Manager Private Certificate Authority (ACM PCA). The PCA must be in the same region as the VPN connection. Public ACM certificates CANNOT be used for VPN authentication. Certificate renewal must be managed before expiry.

## Guardrail 4: Private IP VPN Requires Direct Connect
Private IP VPN connections use private IP addresses and run over AWS Direct Connect. They require a transit gateway with a Direct Connect gateway association. Private IP VPN does NOT work over the public internet. The Direct Connect connection must be active and the DX gateway must be associated with the TGW.

## Guardrail 5: VPN Tunnel Telemetry Is Not Real-Time
VPN tunnel telemetry (VgwTelemetry) shows the last known status of each tunnel. Status updates can be delayed by up to 5 minutes. A tunnel showing UP in telemetry may have gone DOWN recently. Always correlate telemetry with CloudWatch metrics for accurate status.

## Guardrail 6: BGP Dampening Affects Route Convergence
BGP route flapping triggers dampening on the AWS side. Dampened routes are suppressed for increasing periods (up to 60 minutes). Frequent tunnel flapping causes BGP dampening that delays route convergence even after the tunnel stabilizes. Check for dampened routes before concluding BGP is broken.

## Guardrail 7: DPD Settings Must Match Both Sides
Dead Peer Detection (DPD) settings must be compatible between AWS and the customer gateway. AWS uses DPD with a default timeout. If the customer gateway has a shorter DPD timeout, it may declare the tunnel dead before AWS responds. Mismatched DPD causes tunnel flapping.

## Guardrail 8: VPN Throughput Limits Are Per-Tunnel
Each VPN tunnel has a maximum throughput of 1.25 Gbps. This is per-tunnel, not per-connection. A VPN connection with two tunnels can achieve up to 2.5 Gbps with ECMP. Exceeding tunnel throughput causes packet drops, not throttling errors.

## Guardrail 9: VPN Logs Have Delivery Delay
VPN connection logs (when enabled) are delivered to CloudWatch Logs with a delay of up to 5 minutes. Not all tunnel events are logged — only IKE negotiation events, DPD events, and tunnel state changes. Data plane issues (packet loss, latency) are NOT in VPN logs.

## Guardrail 10: Two Tunnels Per Connection for Redundancy
Each VPN connection has exactly two tunnels to different AWS endpoints for redundancy. Both tunnels should be configured on the customer gateway. If only one tunnel is configured, failover will not work. AWS may perform maintenance on one tunnel at a time.

## Guardrail 11: IKE Version Compatibility Matters
AWS supports IKEv1 and IKEv2. The IKE version must match between AWS and the customer gateway. IKEv2 is recommended for better performance and security. Mismatched IKE versions cause tunnel establishment failures that may not have clear error messages.

## Guardrail 12: Static and BGP Routes Have Different Failover Behavior
Static route VPN connections require manual route management and do not automatically failover. BGP-based VPN connections automatically withdraw routes when a tunnel goes down. Do not assume static route VPNs have automatic failover — they require health check-based routing or manual intervention.
