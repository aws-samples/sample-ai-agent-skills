---
name: privatelink-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS PrivateLink and VPC Interface
  Endpoint problems by analyzing endpoint creation, DNS resolution, endpoint status,
  endpoint service configuration, NLB requirements, connection acceptance, connectivity
  failures, security groups, cross-AZ issues, private DNS, split-horizon DNS,
  cross-account access, endpoint policies, throughput, latency, and following
  structured runbooks. Activate when: endpoint creation failures, DNS not resolving,
  endpoint stuck in pending, endpoint service creation issues, NLB not associated,
  connection acceptance problems, connectivity failures through endpoints, security
  group blocking traffic, cross-AZ data transfer concerns, private DNS not working,
  split-horizon DNS issues, cross-account endpoint access denied, endpoint policy
  errors, throughput limitations, latency through endpoints, or the user says
  something is wrong with PrivateLink without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with EC2, ELBv2, IAM, CloudTrail,
  CloudWatch, and optionally Route 53 and VPC Flow Logs permissions.
  Some operations require VPC owner permissions.
---

# AWS PrivateLink / VPC Interface Endpoints Diagnostics

## When to use

Any AWS PrivateLink investigation where the console alone is insufficient — endpoint creation failures, DNS resolution issues, connectivity problems, security group configuration, endpoint service setup, NLB requirements, cross-account access, endpoint policies, private DNS configuration, or performance troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id>
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> --query 'VpcEndpoints[*].{Id:VpcEndpointId,Service:ServiceName,State:State,Type:VpcEndpointType}'
aws ec2 describe-vpc-endpoint-services --service-names <service-name>
aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[*].{Id:VpcId,DnsSupport:EnableDnsSupport,DnsHostnames:EnableDnsHostnames}'
```

### Step 2 — Domain deep dive

```
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].{DnsEntries:DnsEntries,Groups:Groups,SubnetIds:SubnetIds,PrivateDnsEnabled:PrivateDnsEnabled}'
aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id> --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,SubnetId:SubnetId,PrivateIp:PrivateIpAddress,SecurityGroups:Groups}'
aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].{Inbound:IpPermissions,Outbound:IpPermissionsEgress}'
aws ec2 describe-vpc-endpoint-connections --filters Name=vpc-endpoint-id,Values=<endpoint-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20
aws ec2 describe-vpc-endpoint-service-configurations --service-ids <service-id>
aws ec2 describe-vpc-endpoint-service-permissions --service-id <service-id>
aws elbv2 describe-load-balancers --load-balancer-arns <nlb-arn> --query 'LoadBalancers[*].{Name:LoadBalancerName,Type:Type,State:State.Code,Scheme:Scheme}'
version: "1.0.0"
last_updated: "2025-04-12"
```

Read `references/privatelink-guardrails.md` before concluding on any PrivateLink issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-vpc-endpoints` | Check endpoint status, DNS, security groups, subnets |
| `describe-vpc-endpoint-services` | Check available services and service details |
| `describe-vpc-endpoint-service-configurations` | Check endpoint service configuration (provider side) |
| `describe-vpc-endpoint-connections` | Check connection status between endpoint and service |
| `describe-vpc-endpoint-service-permissions` | Check who can create endpoints to a service |
| `describe-network-interfaces` | Check ENIs created by the endpoint |
| `describe-security-groups` | Check security group rules on endpoint ENIs |
| `describe-load-balancers` | Check NLB configuration for endpoint services |
| `describe-vpcs` | Check VPC DNS settings for private DNS |
| VPC Flow Logs | Analyze traffic through endpoint ENIs |

## Gotchas: AWS PrivateLink

- PrivateLink uses interface endpoints, NOT gateway endpoints. Gateway endpoints are for S3 and DynamoDB only and are free. Interface endpoints (PrivateLink) create ENIs in your subnets and have hourly + data processing charges. Do not confuse the two.
- Endpoint services require a Network Load Balancer (NLB). You cannot create an endpoint service with an ALB or CLB. The NLB must be in the same region. The NLB can be internal or internet-facing. Gateway Load Balancers can also be used for endpoint services.
- Private DNS requires enableDnsSupport AND enableDnsHostnames on the VPC. Both VPC attributes must be true for private DNS to work with interface endpoints. Without these, the endpoint's private DNS name will not resolve. Check with describe-vpcs.
- Endpoint policies are resource policies, not IAM policies. Endpoint policies control which AWS principals can use the endpoint to access the service. They are attached to the endpoint itself. The default policy allows full access. Restrictive policies can block legitimate traffic if misconfigured.
- Cross-AZ data transfer costs apply. Each endpoint ENI is in a specific AZ. Traffic crossing AZ boundaries (client in AZ-a, endpoint ENI in AZ-b) incurs cross-AZ data transfer charges. Place endpoint ENIs in the same AZs as your clients.
- One endpoint per VPC per service. You can only create one interface endpoint per VPC for each AWS service. Multiple subnets (AZs) can be associated with a single endpoint. Creating a second endpoint for the same service in the same VPC fails.
- Security groups on endpoint ENIs control access. Interface endpoints have ENIs with security groups. Inbound rules on these security groups must allow traffic from clients. The default security group may not allow the required traffic. Check both client and endpoint security groups.
- Endpoint services need acceptance (manual or auto). By default, endpoint service connections require manual acceptance by the service provider. Auto-accept can be enabled. Pending acceptance connections will not pass traffic. Check connection status.
- DNS resolution returns private IPs only when private DNS is enabled. Without private DNS, the service's default DNS name resolves to public IPs. With private DNS enabled, the same DNS name resolves to the endpoint's private IPs. This enables transparent migration to PrivateLink without application changes.

## Anti-hallucination rules

1. Always cite specific endpoint IDs, ENI IDs, or API responses as evidence.
2. Interface endpoints are NOT gateway endpoints. Never confuse the two types.
3. NLB is required for endpoint services. Never suggest ALB or CLB.
4. Private DNS requires both VPC DNS attributes. Never assume they are enabled.
5. Endpoint policies are resource policies. Never confuse with IAM identity policies.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Endpoint | A1-A3 | Creation failures, DNS resolution, endpoint status |
| B — Service | B1-B3 | Endpoint service creation, NLB requirements, acceptance |
| C — Connectivity | C1-C3 | Connection failures, security groups, cross-AZ |
| D — DNS | D1-D2 | Private DNS, split-horizon DNS |
| E — Cross-Account | E1-E2 | Cross-account access, endpoint policies |
| F — Performance | F1-F2 | Throughput, latency |
| G — Security | G1-G2 | Endpoint policies, security groups |
| Z — Catch-All | Z1 | General troubleshooting |
