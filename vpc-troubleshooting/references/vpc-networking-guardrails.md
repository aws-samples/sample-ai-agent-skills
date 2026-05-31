# VPC Networking Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any VPC networking issue.

## Guardrail 1: Security Groups are Stateful, NACLs are Stateless
Security groups automatically allow return traffic. NACLs do NOT. If you allow inbound TCP/443 in a NACL, you must ALSO allow outbound on ephemeral ports (1024-65535) for the response. This is the #1 NACL misconfiguration.

## Guardrail 2: Security Groups Have ALLOW Rules Only
Security groups cannot deny specific IPs or ports. Default is deny-all-inbound, allow-all-outbound. To block specific traffic, use NACLs. Never suggest adding a deny rule to a security group.

## Guardrail 3: NACL Rules Are Ordered
NACL rules are evaluated in order by rule number (lowest first). First match wins. A deny rule at 50 blocks traffic even if an allow rule exists at 100. Always check rule ordering.

## Guardrail 4: A Subnet is Public Because of Its Route Table
There is no "public subnet" attribute. A subnet is public if its route table has 0.0.0.0/0 → IGW. Private if 0.0.0.0/0 → NAT. Isolated if no default route. The subnet setting "auto-assign public IP" is separate from being a public subnet.

## Guardrail 5: VPC Peering is Non-Transitive
VPC-A peered with VPC-B, and VPC-B peered with VPC-C does NOT mean VPC-A can reach VPC-C. Each pair needs its own peering connection or use Transit Gateway. Never claim traffic can hop through a peered VPC.

## Guardrail 6: 5 Reserved IPs Per Subnet
First 4 IPs and last IP in every subnet are reserved by AWS. A /24 (256 IPs) has 251 usable. A /28 (16 IPs) has only 11 usable. Always account for this when sizing subnets.

## Guardrail 7: NAT Gateway Must Be in a Public Subnet
NAT gateway needs a route to the internet (via IGW) to function. It must be in a subnet with 0.0.0.0/0 → IGW. Private subnets route 0.0.0.0/0 → NAT gateway. Placing NAT in a private subnet breaks internet access.

## Guardrail 8: Flow Logs Don't Show Which Rule Blocked
VPC flow logs show ACCEPT or REJECT at the ENI level. They do NOT indicate whether a security group or NACL caused the REJECT. You must check both independently.

## Guardrail 9: MTU is Clamped Across Boundaries
Jumbo frames (9001 MTU) only work within the same VPC. Traffic through IGW, NAT gateway, VPN, VPC peering, and TGW is clamped to 1500. If DF bit is set, oversized packets are silently dropped.

## Guardrail 10: Gateway Endpoints vs Interface Endpoints
Gateway endpoints (S3, DynamoDB): free, use route table entries, no DNS changes. Interface endpoints: cost per hour + per GB, use ENIs and DNS, require enableDnsHostnames. Never confuse the two types.

## Guardrail 11: Transit Gateway Has Separate Route Tables
TGW route tables are independent of VPC route tables. Both must be configured. A VPC route table needs CIDR → TGW, AND the TGW route table needs CIDR → VPC attachment. Missing either side breaks connectivity.

## Guardrail 12: DNS Requires Both VPC Attributes
enableDnsSupport provides the VPC DNS server (CIDR+2). enableDnsHostnames assigns DNS names to instances and is required for private hosted zones. Both must be true for full DNS functionality.
