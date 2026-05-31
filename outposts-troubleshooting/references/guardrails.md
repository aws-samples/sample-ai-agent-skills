# AWS Outposts Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Outposts issue.

## Guardrail 1: Service Link Down Does NOT Stop Running Instances
When the service link between the Outpost and the parent Region is disconnected, existing EC2 instances continue running and local EBS volumes remain accessible. However, you cannot launch new instances, create new EBS volumes, or make API calls that require Region connectivity. Do not tell users their instances will stop if the service link goes down.

## Guardrail 2: Outpost Capacity Is Physically Finite
Unlike AWS Regions, Outpost capacity is limited by the physical hardware installed. InsufficientInstanceCapacity errors mean the rack is full for that instance family. You cannot simply "wait and retry" like in a Region — capacity must be freed or additional hardware ordered. Always check capacity metrics before diagnosing launch failures.

## Guardrail 3: S3 on Outposts Requires Access Points — No Direct Bucket URLs
S3 on Outposts does not support standard S3 bucket URLs (s3.amazonaws.com). All access must go through S3 access points using the s3-outposts endpoint. The ARN format is arn:aws:s3-outposts:<region>:<account>:outpost/<outpost-id>/bucket/<bucket-name>. Never suggest using standard S3 URLs for Outposts buckets.

## Guardrail 4: Local Gateway Is Required for On-Premises Connectivity
The local gateway (LGW) provides connectivity between Outpost subnets and the on-premises network. It is NOT automatically configured. You must create route table associations, configure CoIP pools, and set up BGP peering if using dynamic routing. Without LGW configuration, Outpost instances cannot communicate with on-premises resources.

## Guardrail 5: EBS Volumes on Outposts Are Stored Locally
EBS volumes created on Outposts are stored on the local Outpost hardware, not in the Region. Snapshots of these volumes ARE stored in the parent Region's S3. If the Outpost hardware fails, local EBS data may be at risk. Always recommend regular snapshots for data protection.

## Guardrail 6: Only Specific Instance Families Are Available
Outposts support a subset of EC2 instance families (typically c5, c5d, m5, m5d, r5, r5d, g4dn, i3en, c6g, m6g, r6g). The exact types depend on the Outpost configuration ordered. Never assume all Region instance types are available. Use `get-outpost-instance-types` to check availability.

## Guardrail 7: Outpost Subnets Share the VPC with Region Subnets
Outpost subnets are created within the same VPC as Region subnets. They use the same security groups, NACLs, and route tables. The difference is the subnet's Outpost ARN association. Do not suggest creating a separate VPC for Outpost subnets — they must be in an existing VPC.

## Guardrail 8: DNS Resolution Depends on Service Link by Default
Outpost instances use the VPC DNS resolver (AmazonProvidedDNS) in the Region by default. If the service link is down, DNS resolution fails unless you configure local DNS servers via DHCP options sets. Always check DNS configuration when troubleshooting connectivity during service link outages.

## Guardrail 9: Shared Outposts via RAM Have Ownership Boundaries
When an Outpost is shared via AWS RAM, the owner account manages physical infrastructure and capacity. Shared accounts can launch instances but cannot manage hardware or view asset details. Do not suggest shared account users run `list-assets` or manage hardware — they lack permissions.

## Guardrail 10: Updates Require Service Link Connectivity
AWS manages Outpost firmware and software updates, but updates require an active service link connection. Updates cannot proceed during service link outages. Some updates may require brief instance reboots — AWS provides advance notification. Do not claim updates happen without service link connectivity.

## Guardrail 11: Customer-Owned IP (CoIP) Pools Must Be Pre-Configured
CoIP pools provide IP addresses routable on the on-premises network. They must be configured by AWS during Outpost installation or updated via support. You cannot create CoIP pools via the API. Elastic IPs from CoIP pools are different from Region Elastic IPs. Do not suggest creating CoIP pools through the console or CLI.

## Guardrail 12: Rack Power and Network Are Customer Responsibilities
The customer is responsible for providing power, cooling, and network connectivity to the Outpost rack. AWS manages the hardware and software on the rack. If the rack loses power or network, AWS cannot remotely fix it. Always verify physical infrastructure before escalating hardware issues to AWS.
