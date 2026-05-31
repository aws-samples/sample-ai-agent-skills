# Direct Connect & Site-to-Site VPN Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Direct Connect or VPN issue.

## Guardrail 1: DX Is NOT Encrypted by Default
Direct Connect provides a dedicated network connection but does NOT encrypt traffic. Use MACsec (802.1AE) for Layer 2 encryption on 10 Gbps and 100 Gbps connections, or establish a VPN tunnel over the DX connection for IPsec encryption.

## Guardrail 2: Both VPN Tunnels Must Be UP
Each Site-to-Site VPN connection has two tunnels for redundancy. AWS performs maintenance on one tunnel at a time. If only one tunnel is configured on the customer gateway, connectivity is lost during maintenance. Always configure both tunnels.

## Guardrail 3: VPN Throughput Is 1.25 Gbps Per Tunnel
Each VPN tunnel has a hard limit of 1.25 Gbps. This cannot be increased. For higher throughput, use multiple VPN connections with ECMP on Transit Gateway, or use Direct Connect.

## Guardrail 4: BGP Requires Matching Configuration on Both Sides
BGP session establishment requires matching ASN configuration, correct BGP authentication key, proper VLAN tagging, and correct peer IP addresses. AWS provides the peer IPs — the customer router must peer with the AWS-side address.

## Guardrail 5: LAG Members Must Have Same Speed and Location
All connections in a Link Aggregation Group must have the same bandwidth (e.g., all 10 Gbps) and terminate at the same AWS Direct Connect location. Mixing speeds or locations causes LAG creation to fail.

## Guardrail 6: Private VIF Connects to One VPC, Transit VIF to TGW
A Private VIF connects to a single VPC via a Virtual Private Gateway (or multiple VPCs via DX Gateway). A Transit VIF connects to a Transit Gateway via DX Gateway for scalable multi-VPC routing. Do not confuse the two.

## Guardrail 7: DX Gateway Is Required for Multi-Region
A Direct Connect Gateway is required to connect a DX connection to VPCs in multiple regions. Without a DX Gateway, each VPC in a different region would need its own physical DX connection.

## Guardrail 8: VPN Over DX Uses Public VIF
To run an IPsec VPN tunnel over a Direct Connect connection, you must use a Public VIF to reach the AWS VPN endpoint public IPs. A Private VIF cannot be used for this purpose.

## Guardrail 9: Route Propagation Must Be Enabled on VPC Route Tables
VGW route propagation must be explicitly enabled on each VPC route table that needs to receive routes from DX or VPN. Routes are not automatically added to all route tables.

## Guardrail 10: DX Failover to VPN Requires BGP Tuning
For DX-to-VPN failover, the VPN path must be less preferred than DX. Use AS path prepending on the VPN route or set lower local preference. Without BGP tuning, traffic may prefer VPN over DX or fail to fail over.

## Guardrail 11: MACsec Is Only Available on Dedicated Connections
MACsec encryption is only supported on dedicated Direct Connect connections of 10 Gbps and 100 Gbps. It is not available on hosted connections or sub-1 Gbps connections. Both ends must support MACsec.

## Guardrail 12: VPN Tunnel Idle Timeout Can Cause Drops
AWS VPN tunnels may drop if idle for more than 10 seconds (DPD timeout). Configure Dead Peer Detection (DPD) on the customer gateway and use a keep-alive mechanism (e.g., ping) to maintain the tunnel during idle periods.
