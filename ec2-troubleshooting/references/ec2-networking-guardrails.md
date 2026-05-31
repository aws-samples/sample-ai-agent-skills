# EC2 Networking Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EC2 networking issue.

## Guardrail 1: Security Groups are Stateful, NACLs are Stateless
Security groups automatically allow return traffic for allowed connections. NACLs do NOT — you must explicitly allow both directions. If outbound works but responses don't come back, check NACLs for ephemeral port (1024-65535) rules.

## Guardrail 2: Security Groups Have No Deny Rules
You CANNOT block specific IPs with security groups. SGs only have ALLOW rules. Use NACLs for deny rules. Never suggest adding a "deny rule" to a security group.

## Guardrail 3: Source/Destination Check
By default, EC2 checks that traffic source/destination matches the instance's IP. This MUST be disabled for: NAT instances, load balancers, VPN instances, or any traffic forwarding. Forgetting this is a common cause of "traffic not flowing through my NAT instance."

## Guardrail 4: Public IP vs Elastic IP
Auto-assigned public IPs change on stop+start. Elastic IPs are static. If an application hardcodes a public IP and the instance is stopped+started, connectivity breaks. Always use EIPs or DNS names for stable addressing.

## Guardrail 5: VPC Flow Logs Show Direction, Not Content
Flow logs show ACCEPT/REJECT at the ENI level. They do NOT show packet content, application data, or which specific rule caused the action. Use flow logs to confirm traffic is reaching/leaving the ENI, then check SG/NACL rules for the specific block.

## Guardrail 6: MTU and Jumbo Frames
VPC default MTU is 1500. Jumbo frames (9001 MTU) only work within the same VPC and between instances that support it. Traffic through IGW, NAT gateway, VPN, or VPC peering is clamped to 1500. Path MTU Discovery (PMTUD) requires ICMP type 3 code 4 to be allowed.

## Guardrail 7: DNS in VPC
VPC DNS resolver is at VPC CIDR base +2 (e.g., 10.0.0.2 for 10.0.0.0/16). Requires enableDnsSupport=true. Private hosted zone resolution requires enableDnsHostnames=true. Custom DHCP option sets can override the DNS server — check this if DNS fails.

## Guardrail 8: Cross-AZ Traffic Costs and Latency
Traffic between AZs incurs data transfer charges and adds ~1ms latency. Same-AZ traffic is free (within VPC) and lower latency. For latency-sensitive applications, keep communicating instances in the same AZ.

## Guardrail 9: ENI Limits are Instance-Type Specific
Each instance type has a maximum number of ENIs and IPs per ENI. These limits are fixed and cannot be increased. Check `describe-instance-types` for the specific limits. On pre-Gen7 instances, ENIs share attachment slots with EBS volumes.

## Guardrail 10: VPC Peering is Non-Transitive
If VPC-A peers with VPC-B and VPC-B peers with VPC-C, VPC-A CANNOT reach VPC-C through VPC-B. Use Transit Gateway for transitive routing. Also, peering does not support overlapping CIDRs.

## Guardrail 11: ENA Enhanced Networking
Nitro instances require ENA (Elastic Network Adapter). The AMI must have ENA support enabled AND the ENA driver installed. Launching a Nitro instance with a non-ENA AMI will fail or have no network connectivity.

## Guardrail 12: IMDS Hop Limit
Default hop limit is 1 (IMDSv2) or 1 (configurable). Containers add an extra network hop. If hop limit is 1, containers cannot reach IMDS. Set hop limit to 2 for containerized workloads. This affects credential delivery, not just metadata.
