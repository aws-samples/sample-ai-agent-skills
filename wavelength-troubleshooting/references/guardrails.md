# AWS Wavelength Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Wavelength issue.

## Guardrail 1: Wavelength Zones Must Be Explicitly Opted-In
Wavelength Zones are not enabled by default. Each account must opt-in to each Wavelength Zone individually. If a zone is not opted-in, subnets cannot be created and instances cannot be launched. Always verify opt-in status before diagnosing deployment failures.

## Guardrail 2: Carrier Gateway Is Required for Mobile Network Traffic
The carrier gateway provides connectivity between Wavelength Zone instances and the carrier's mobile network. It is NOT an internet gateway or NAT gateway. Without a carrier gateway, instances cannot receive traffic from mobile devices on the carrier network. Each VPC can have one carrier gateway.

## Guardrail 3: Carrier IPs Are NOT Elastic IPs
Carrier IPs are allocated from the carrier's IP address pool and are routable on the carrier's 5G/LTE network. They are different from AWS Elastic IPs. Carrier IPs are associated with instances in Wavelength Zones via the carrier gateway. Do not confuse carrier IP allocation with standard EIP allocation.

## Guardrail 4: Limited Instance Types in Wavelength Zones
Only specific instance families are available in Wavelength Zones (typically t3, r5, g4dn). The exact types vary by Wavelength Zone and carrier. Never assume all Region instance types are available. Use `describe-instance-type-offerings --location-type availability-zone --filters Name=location,Values=<wl-zone>` to check.

## Guardrail 5: No Internet Gateway or NAT Gateway in Wavelength Zones
Internet gateways and NAT gateways are NOT supported in Wavelength Zones. For carrier network traffic, use the carrier gateway. For internet access, route traffic through the parent Region VPC via a NAT gateway or internet gateway in a Region subnet. Never suggest attaching an IGW to a Wavelength subnet.

## Guardrail 6: EBS Volumes Are Stored Locally in the Wavelength Zone
EBS volumes created in Wavelength Zones are stored on local infrastructure, not in the Region. Snapshots are stored in the parent Region's S3. If the Wavelength Zone infrastructure has issues, local EBS data may be at risk. Always recommend regular snapshots.

## Guardrail 7: S3 Access Requires Routing Through the Region
There are no S3 endpoints directly in Wavelength Zones. To access S3, traffic must route through the parent Region VPC. Use S3 gateway endpoints in the Region VPC for optimal performance. Direct S3 access from Wavelength adds latency due to the Region hop.

## Guardrail 8: DNS Resolution Uses the Region VPC Resolver
Wavelength Zone instances use the VPC DNS resolver in the parent Region. DNS queries traverse the carrier-to-Region path, adding some latency. For latency-sensitive applications, consider local DNS caching on the instance.

## Guardrail 9: Auto Scaling Groups Require Wavelength-Specific Configuration
Auto Scaling groups can include Wavelength Zone subnets, but launch templates must specify Wavelength-compatible instance types. Mixed instance policies must only include types available in the Wavelength Zone. Scaling across Region and Wavelength subnets requires careful configuration.

## Guardrail 10: Not All AWS Services Are Available in Wavelength Zones
Only a subset of AWS services run directly in Wavelength Zones (EC2, EBS, ECS, EKS, IAM, CloudWatch, CloudFormation). Other services must be accessed through the Region. Do not suggest deploying RDS, ElastiCache, or other unsupported services directly in Wavelength Zones.

## Guardrail 11: Wavelength Subnets Are in the Same VPC as Region Subnets
Wavelength subnets are created in the same VPC as Region subnets. They share the same VPC CIDR, security groups, and NACLs. The difference is the availability zone type (wavelength-zone). Route tables for Wavelength subnets should route to the carrier gateway, not an internet gateway.

## Guardrail 12: Carrier Network Conditions Affect Performance
Latency and throughput in Wavelength Zones depend on the carrier's network conditions, not just AWS infrastructure. Network congestion, signal strength, and carrier routing affect end-to-end performance. Do not guarantee specific latency numbers — they vary by carrier, location, and network load.
