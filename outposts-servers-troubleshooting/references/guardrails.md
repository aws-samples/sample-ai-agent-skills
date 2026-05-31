# AWS Outposts Servers Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Outposts Servers issue.

## Guardrail 1: Servers Are Not Racks
Outposts Servers (1U/2U) are single-server deployments for edge locations. They are NOT the same as Outposts racks (42U). Servers support fewer instance types, have no local redundancy, and have different capacity limits. Never apply rack documentation to server form factors.

## Guardrail 2: Service Link Is Required for Control Plane
The service link connects the Outposts Server to the parent AWS Region. Without it, new instance launches, EBS volume creation, and API calls fail. Running instances continue to operate during brief disconnections. Do not assume full offline operation.

## Guardrail 3: EBS Local Volumes Are Not Regionally Durable
EBS volumes on Outposts Servers are stored on local NVMe drives. They do NOT replicate to the AWS Region. If the server hardware fails, local EBS data is lost. Do not claim the same durability as regional EBS.

## Guardrail 4: Limited Instance Type Support
Outposts Servers support only specific instance families: Graviton-based (c6gd, m6gd, r6gd) and Intel-based (c6id, m6id, r6id). Not all sizes within these families may be available. Always verify with `get-outpost-instance-types`.

## Guardrail 5: Single Point of Failure
A single Outposts Server has no hardware redundancy. A server failure affects ALL workloads. Design for resilience at the application layer or deploy multiple servers. Do not claim built-in high availability.

## Guardrail 6: Customer-Owned IP Pools Are Required
Outposts Servers use customer-owned IP (CoIP) pools for local network addressing. These must be configured correctly in the local gateway route table. Misconfigured CoIP pools prevent instance launches and network connectivity.

## Guardrail 7: Software Updates Require Service Link
AWS manages software updates for Outposts Servers, but updates require an active service link to the parent Region. Updates cannot be applied while the server is disconnected. Do not suggest manual firmware updates.

## Guardrail 8: Subnet Must Be Associated with Outpost
EC2 instances on Outposts must be launched in a subnet associated with the Outpost. A regular VPC subnet will launch instances in the Region, not on the Outpost. Verify subnet-to-Outpost association before diagnosing launch failures.

## Guardrail 9: Local Gateway Handles On-Premises Routing
The local gateway (LGW) provides connectivity between the Outpost and the on-premises network. LGW route tables, VIF groups, and CoIP associations must all be correctly configured. Do not confuse LGW with internet gateway or NAT gateway.

## Guardrail 10: Activation Requires Physical and Network Prerequisites
Server activation requires physical installation, network connectivity (service link), and proper AWS account configuration. Activation failures can be physical (power, cabling), network (firewall, DNS), or account-level (permissions, site configuration).

## Guardrail 11: Outpost Site Configuration Is Separate from Outpost
An Outpost site represents the physical location. The Outpost resource represents the hardware. Sites must be created before Outposts can be ordered. Do not confuse site-level issues with Outpost-level issues.

## Guardrail 12: CloudWatch Metrics Require Service Link
CloudWatch metrics for Outposts Servers are published via the service link. During disconnection, metrics are not available in CloudWatch. Local monitoring alternatives should be configured for disconnected scenarios.
