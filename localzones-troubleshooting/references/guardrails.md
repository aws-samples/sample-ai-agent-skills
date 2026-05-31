# AWS Local Zones Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Local Zones issue.

## Guardrail 1: Local Zones Require Explicit Opt-In
Local Zones are NOT enabled by default. You must opt in to each Local Zone group using `aws ec2 modify-availability-zone-group`. Without opt-in, the zone does not appear in subnet creation or instance launch options.

## Guardrail 2: Instance Type Availability Varies by Zone
Each Local Zone supports a different subset of instance types. Do not assume an instance type available in the parent region is available in the Local Zone. Always check with `describe-instance-type-offerings` filtered by location.

## Guardrail 3: Local Zones Are Not Wavelength Zones
Local Zones extend a region to a metro area for low-latency access. Wavelength Zones are embedded in 5G carrier networks. They have different use cases, different available services, and different networking models. Do not confuse them.

## Guardrail 4: EBS Volume Types Are Limited
Not all EBS volume types are available in Local Zones. Typically gp2 and gp3 are supported. io1, io2, sc1, st1, and magnetic may not be available. Check volume type availability before creating volumes.

## Guardrail 5: Not All AWS Services Are Available
Local Zones support a subset of AWS services. EC2, EBS, VPC, ELB, and some RDS engines are commonly available. Services like Lambda, ECS Fargate, S3 (as a local service), and many others are NOT available in Local Zones.

## Guardrail 6: Networking Between Local Zone and Parent Region
Traffic between Local Zone subnets and parent region subnets traverses the network path between the Local Zone and the region. This adds latency compared to intra-region traffic. For lowest latency, keep communicating resources in the same Local Zone.

## Guardrail 7: RDS Support Is Limited
RDS in Local Zones supports specific DB engines (MySQL, PostgreSQL) and limited instance classes. Multi-AZ deployments are NOT supported in Local Zones. Read replicas may have limitations. Always verify engine and instance class availability.

## Guardrail 8: Local Zone Pricing Differs from Parent Region
EC2 instances, EBS volumes, and data transfer in Local Zones may have different pricing than the parent region. Do not assume parent region pricing applies. Check Local Zone-specific pricing before deploying.

## Guardrail 9: Internet Gateway Is Shared with Parent Region VPC
Local Zone subnets use the same internet gateway as the parent region VPC. There is no separate internet gateway for Local Zones. Route tables for Local Zone subnets must route internet traffic through the VPC's internet gateway.

## Guardrail 10: Capacity Is More Constrained Than Regions
Local Zones have less capacity than full regions. Instance launches may fail with InsufficientInstanceCapacity more frequently. Plan for capacity constraints and consider using Capacity Reservations where supported.

## Guardrail 11: Subnet CIDR Must Come from VPC CIDR
Local Zone subnets are created within an existing VPC. The subnet CIDR must be within the VPC CIDR range. You cannot create a VPC in a Local Zone — you extend an existing VPC by creating subnets in the Local Zone.

## Guardrail 12: Auto Scaling Groups Can Span Local Zones
Auto Scaling groups can include Local Zone subnets, but instance type availability and capacity constraints may cause scaling failures. Configure multiple instance types in mixed instances policy for better availability.
