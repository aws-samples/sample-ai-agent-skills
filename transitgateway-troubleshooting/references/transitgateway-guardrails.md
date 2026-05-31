# AWS Transit Gateway Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Transit Gateway issue.

## Guardrail 1: TGW Route Tables Are Separate from VPC Route Tables
TGW route tables and VPC route tables are completely independent. A VPC route table entry pointing to a TGW only directs traffic TO the TGW. The TGW route table then determines where traffic goes NEXT. Both must be configured correctly for end-to-end connectivity. Never confuse a TGW route table ID (tgw-rtb-*) with a VPC route table ID (rtb-*).

## Guardrail 2: Attachments Need a Subnet in Each AZ
VPC attachments require a subnet in each AZ where you want connectivity. If a VPC attachment only has a subnet in AZ-a, instances in AZ-b of that VPC cannot reach the TGW directly. Traffic from AZ-b must cross AZ boundaries within the VPC to reach the TGW attachment subnet. Always specify subnets in all AZs where instances reside.

## Guardrail 3: Route Propagation vs Static Routes
Propagated routes are automatically learned from attachments (VPC CIDRs, VPN routes via BGP). Static routes are manually configured and take precedence over propagated routes for the same CIDR. When both exist for the same destination, the static route wins. Mixing both without understanding precedence causes routing confusion.

## Guardrail 4: Blackhole Routes Block Traffic Silently
Blackhole routes drop traffic without any error or notification. They appear when an attachment is deleted but the route remains, or when manually created for traffic isolation. Always search for blackhole routes when debugging connectivity: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rtb-id> --filters Name=state,Values=blackhole`.

## Guardrail 5: Inter-Region Peering Does NOT Support Route Propagation
Inter-region TGW peering attachments do not support automatic route propagation. You MUST create static routes pointing to the peering attachment for each CIDR you want reachable across regions. Peering also does not support multicast or Connect attachments. Bandwidth is limited and varies by region pair.

## Guardrail 6: Each Attachment Associates with Exactly ONE Route Table
An attachment can be associated with only one TGW route table at a time. Association determines which route table is used for routing decisions for traffic FROM that attachment. A route table can have many attachments associated with it. Changing association requires disassociating from the current table first.

## Guardrail 7: Appliance Mode for Stateful Inspection
When routing traffic through a security appliance (firewall, IDS/IPS) in a VPC, enable appliance mode on the VPC attachment. Without appliance mode, return traffic may use a different AZ path than the original traffic, breaking stateful inspection. Appliance mode ensures symmetric routing through the same AZ.

## Guardrail 8: MTU Considerations Across Attachment Types
VPC attachments support MTU 8500 within the same region. VPN attachments support MTU 1500. Inter-region peering supports MTU 8500. Mismatched MTU between source and destination causes packet drops for large packets. Use path MTU discovery or set MTU explicitly on instances.

## Guardrail 9: Cross-Account Sharing via RAM
TGW sharing via AWS Resource Access Manager (RAM) requires the shared account to accept the resource share. The shared account can create attachments to the TGW but cannot modify the TGW itself, its route tables, or other accounts' attachments. The TGW owner manages all route tables and routing decisions.

## Guardrail 10: Default Route Table Auto-Association and Auto-Propagation
TGW has a default route table that auto-associates and auto-propagates new attachments unless disabled. If auto-propagation is on, all attachments share routes through the default table, creating a flat network. Disable auto-association and auto-propagation for network segmentation. Check with describe-transit-gateways Options field.

## Guardrail 11: Multicast Must Be Enabled at TGW Creation
Multicast support must be enabled when creating the TGW — it cannot be enabled on an existing TGW. If multicast is needed and the TGW was created without it, you must create a new TGW with multicast enabled. Not all instance types support multicast. Check the MulticastSupport field in describe-transit-gateways.

## Guardrail 12: Attachment Lifecycle States Affect Connectivity
TGW attachments go through lifecycle states: initiatingRequest → pendingAcceptance → pending → available → modifying → deleting → deleted. Also: rejecting, rejected, failing, failed, rollingBack. Only attachments in "available" state pass traffic. Cross-account attachments require acceptance. Check the State field in describe-transit-gateway-attachments.
