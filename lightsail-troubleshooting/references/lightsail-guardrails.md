# Amazon Lightsail Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Lightsail issue.

## Guardrail 1: Lightsail Firewall Is NOT VPC Security Groups
Lightsail instances have a built-in firewall configured per-instance that controls inbound traffic only. This is separate from VPC security groups. Default rules allow SSH (22) and HTTP (80). Additional ports must be explicitly opened. The firewall is managed through the Lightsail API/console, not EC2 security group APIs. Outbound traffic is not restricted by the Lightsail firewall.

## Guardrail 2: Static IPs Incur Charges When Not Attached to Running Instances
Static IPs are free ONLY when attached to a running Lightsail instance. Unattached static IPs or static IPs on stopped instances cost $0.005/hour. Each account can have up to 5 static IPs per region. Static IPs persist across instance stop/start. Releasing a static IP makes the address available to other customers.

## Guardrail 3: Lightsail Managed Databases Are NOT RDS
Lightsail databases support MySQL and PostgreSQL only. They do NOT support Multi-AZ deployments, read replicas, IAM database authentication, Performance Insights, or Enhanced Monitoring. Backup retention is 7 days (automatic). Point-in-time restore is supported. Do not suggest RDS-specific features for Lightsail databases.

## Guardrail 4: Container Service Sizes Are Fixed Tiers
Container services use predefined power levels: Nano (0.25 vCPU, 512 MB), Micro (0.5 vCPU, 1 GB), Small (1 vCPU, 2 GB), Medium (2 vCPU, 4 GB), Large (4 vCPU, 8 GB), XLarge (8 vCPU, 16 GB). You cannot specify custom CPU or memory values. Scale is 1-20 nodes per service. Container images must be pushed to the Lightsail container image registry or a public registry.

## Guardrail 5: VPC Peering Is One-Way from Lightsail to VPC
Lightsail VPC peering allows Lightsail instances to access VPC resources using private IPs. However, VPC resources CANNOT initiate connections to Lightsail instances through the peering connection. Peering must be enabled in the Lightsail console (Account settings). Peering connects to the default VPC in the same region.

## Guardrail 6: Snapshots Are the Primary Backup Mechanism
Instance snapshots capture the entire instance including OS, applications, and data. Disk snapshots capture additional attached block storage disks only. Snapshots are stored in S3 (managed by Lightsail). Snapshots can create new instances in the same or different region. Automatic snapshots can be enabled per-instance with a configurable time.

## Guardrail 7: Lightsail DNS Has Limitations Compared to Route 53
Lightsail DNS zones are free and support A, AAAA, CNAME, MX, NS, SRV, and TXT records. They do NOT support alias records, weighted routing, latency-based routing, geolocation routing, or health checks. For advanced DNS features, use Route 53. Lightsail DNS zones can coexist with Route 53 hosted zones.

## Guardrail 8: Instance Plans Define Resource Limits
Each Lightsail plan includes fixed CPU, memory, storage, and data transfer. Exceeding the data transfer allowance incurs overage charges. CPU performance may be burstable on smaller plans (similar to T-class EC2 instances). Sustained high CPU usage on burstable plans results in throttling.

## Guardrail 9: Bundle Migration Has Constraints
You can only upgrade to a larger bundle, not downgrade. To change bundles, create a snapshot and launch a new instance from the snapshot with the desired bundle. The old instance must be manually deleted. IP addresses change unless you use a static IP. Data transfer allowances differ between bundles.

## Guardrail 10: Automatic Snapshots Have a Fixed Retention
Automatic snapshots retain the most recent 7 snapshots. Older automatic snapshots are deleted automatically. You can convert an automatic snapshot to a manual snapshot to preserve it. Automatic snapshot time is configurable but runs once per day. Manual snapshots have no automatic retention limit.

## Guardrail 11: Lightsail Is Simpler but More Limited Than EC2
Lightsail provides a simplified interface but lacks many EC2 features: no instance store, no placement groups, no dedicated hosts, limited instance types, no Spot instances, no Reserved instances. For workloads requiring these features, migrate to EC2. Lightsail instances can be exported to EC2 via snapshot.

## Guardrail 12: Load Balancer Health Checks Affect Instance Routing
Lightsail load balancers perform health checks on a configurable path. Instances failing health checks are removed from rotation. The default health check path is /. Custom health check paths must return HTTP 200. Health check interval is fixed at 30 seconds. Unhealthy threshold is 2 consecutive failures.
