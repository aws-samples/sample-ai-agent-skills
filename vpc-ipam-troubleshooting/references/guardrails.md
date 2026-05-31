# VPC IPAM Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any VPC IPAM issue.

## Guardrail 1: IPAM Is Organization-Level
IPAM is designed to manage IP addresses across an entire AWS Organization. A single IPAM can span multiple accounts and regions. Do not create separate IPAMs per account unless there is a specific isolation requirement.

## Guardrail 2: Pool Locale Restricts Allocation Region
Each IPAM pool has a locale (region) that determines where allocations can be made. A pool with locale us-east-1 can only allocate CIDRs for resources in us-east-1. For multi-region deployments, create regional pools under a top-level pool.

## Guardrail 3: CIDR Provisioning Is Asynchronous
After provisioning a CIDR to a pool, the state transitions from pending-provision to provisioned. Allocations will fail if attempted before the CIDR is fully provisioned. Always verify the CIDR state before allocating.

## Guardrail 4: Two Default Scopes Exist
Every IPAM has two default scopes: public (for public IP space) and private (for private IP space like RFC 1918). Pools must be created in the appropriate scope. Do not mix public and private CIDRs in the wrong scope.

## Guardrail 5: Resource Discovery Required for Cross-Account
IPAM resource discovery must be configured and shared via RAM for IPAM to see resources in member accounts. Without resource discovery, IPAM only sees resources in the IPAM owner account.

## Guardrail 6: Allocation Netmask Length Matters
When allocating from a pool, the requested netmask length must be within the pool's allocation rules (min/max netmask length). Requesting a /16 from a pool that only allows /24 to /28 will fail.

## Guardrail 7: CIDR Overlap Detection Is Not Real-Time
IPAM detects CIDR overlaps through periodic resource discovery scans. New overlaps may not be detected immediately. Do not assume the absence of overlap alerts means no overlaps exist.

## Guardrail 8: BYOIP Requires RIR Verification
Bringing your own IP addresses (BYOIP) requires verification with the Regional Internet Registry (ARIN, RIPE, APNIC). This process can take days. Do not expect immediate BYOIP provisioning.

## Guardrail 9: Pool Hierarchy Affects Allocation
IPAM pools can be nested (parent-child). Child pools allocate from parent pool CIDRs. Deleting a parent pool affects all child pools. Understand the pool hierarchy before making changes.

## Guardrail 10: IPAM Operating Regions Must Be Configured
IPAM must be configured with operating regions to manage resources in those regions. If a region is not in the operating regions list, IPAM cannot discover or manage resources there.

## Guardrail 11: Public IP Insights Requires IPAM
Public IP Insights is a feature of IPAM that provides visibility into public IP usage. It requires an IPAM to be created and configured. It is not a standalone feature.

## Guardrail 12: Compliance Status Depends on Pool Rules
IPAM compliance monitoring checks if VPC CIDRs are allocated from IPAM pools. VPCs with manually assigned CIDRs (not from IPAM) will show as non-compliant. This is expected behavior, not an error.
