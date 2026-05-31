# Amazon Route 53 Resolver Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Route 53 Resolver issue.

## Guardrail 1: Inbound vs Outbound Direction
Inbound endpoints receive DNS queries FROM on-premises or external networks INTO the VPC. Outbound endpoints forward DNS queries FROM the VPC TO on-premises or external DNS servers. The direction is from the VPC's perspective. Never reverse the direction.

## Guardrail 2: Endpoints Require Multi-AZ IP Addresses
Resolver endpoints require at least 2 IP addresses in different Availability Zones for high availability. Each IP address consumes an ENI in the subnet. Ensure subnets have available IP addresses and ENI capacity.

## Guardrail 3: Forwarding Rules vs DNS Firewall Rules Are Different
Forwarding rules control WHERE DNS queries are sent (to specific DNS servers). DNS Firewall rules control WHETHER DNS queries are allowed or blocked based on domain names. They are separate features with different configuration and evaluation paths.

## Guardrail 4: System Rules Take Precedence
System rules (auto-created for AWS internal domains like amazonaws.com, compute.internal) take precedence over custom forwarding rules. You cannot override system rules. Queries for AWS internal domains always resolve via the VPC DNS resolver.

## Guardrail 5: DNS Firewall Priority Order Matters
DNS Firewall rules within a rule group are evaluated by priority number (lowest first). The first matching rule determines the action. A BLOCK at priority 100 takes effect before an ALLOW at priority 200. Plan priority numbers carefully.

## Guardrail 6: Query Logging Captures All VPC DNS Queries
Query logging captures ALL DNS queries made in the VPC, not just those going through resolver endpoints. This includes queries to AmazonProvidedDNS. Log volume can be very high in busy VPCs — plan storage and costs accordingly.

## Guardrail 7: DNSSEC Validation Can Break Resolution
Enabling DNSSEC validation causes queries for domains with invalid or missing DNSSEC signatures to return SERVFAIL instead of the DNS response. This can break resolution for domains with misconfigured DNSSEC. Test thoroughly before enabling in production.

## Guardrail 8: RAM Sharing Requires Organization Membership
Sharing resolver rules and DNS Firewall rule groups via RAM requires both accounts to be in the same AWS Organization (by default). External sharing must be explicitly enabled in RAM settings. Shared resources appear in the recipient account but are managed by the owner.

## Guardrail 9: VPC Association Is Required for Rules to Take Effect
Forwarding rules and DNS Firewall rule groups must be associated with a VPC to take effect. Creating a rule without associating it to a VPC has no impact on DNS resolution. Verify associations after creating rules.

## Guardrail 10: Endpoint Security Groups Control Access
Resolver endpoint ENIs have security groups that control network access. Inbound endpoints need inbound DNS (TCP/UDP 53) from on-premises networks. Outbound endpoints need outbound DNS (TCP/UDP 53) to target DNS servers. Incorrect security groups are a common cause of connectivity failures.

## Guardrail 11: Resolver Endpoints Are Regional
Resolver endpoints are regional resources. They only handle DNS queries within the VPC and region where they are created. Cross-region DNS resolution requires endpoints in each region or Transit Gateway DNS support.

## Guardrail 12: DNS Firewall Fail-Open vs Fail-Close
DNS Firewall has a fail-open/fail-close configuration per VPC. Fail-open allows DNS queries when the firewall is unavailable. Fail-close blocks all DNS queries when the firewall is unavailable. The default is fail-open. Understand the implications before changing.
