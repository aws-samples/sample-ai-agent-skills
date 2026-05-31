# AWS PrivateLink Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS PrivateLink issue.

## Guardrail 1: Interface Endpoints Are NOT Gateway Endpoints
Interface endpoints (PrivateLink) create ENIs in your subnets with private IPs. Gateway endpoints (S3, DynamoDB only) modify route tables and are free. Do not confuse the two. Interface endpoints have hourly charges + data processing fees. Gateway endpoints have no charges. Interface endpoints support security groups; gateway endpoints use endpoint policies only.

## Guardrail 2: Endpoint Services Require a Network Load Balancer (NLB)
You cannot create a VPC endpoint service with an ALB, CLB, or any other load balancer type. Only NLB (or Gateway Load Balancer for GWLB endpoints) is supported. The NLB must be in the same region as the endpoint service. The NLB can be internal or internet-facing. Multiple NLBs can be associated with a single endpoint service for multi-AZ.

## Guardrail 3: Private DNS Requires Both VPC DNS Attributes Enabled
For private DNS names to work with interface endpoints, the VPC must have BOTH enableDnsSupport=true AND enableDnsHostnames=true. Without these, the endpoint's private DNS name will not resolve to the endpoint's private IPs. Check with: aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport.

## Guardrail 4: Endpoint Policies Are Resource Policies, Not IAM Identity Policies
Endpoint policies are JSON resource policies attached to the VPC endpoint. They control which principals can use the endpoint to access the service. The default policy allows full access (all principals, all actions). Endpoint policies work IN ADDITION to IAM policies — both must allow the action. An endpoint policy cannot grant access that IAM denies.

## Guardrail 5: Cross-AZ Data Transfer Costs Apply to Endpoint Traffic
Each interface endpoint ENI exists in a specific AZ. If a client in AZ-a sends traffic through an endpoint ENI in AZ-b, cross-AZ data transfer charges apply. To minimize costs, create endpoint ENIs in all AZs where clients reside. Use AZ-specific DNS names (if available) to route traffic to the local AZ endpoint ENI.

## Guardrail 6: One Interface Endpoint Per VPC Per Service
You can only create one interface endpoint per VPC for each AWS service or endpoint service. Attempting to create a second endpoint for the same service in the same VPC will fail. A single endpoint can span multiple subnets (one per AZ). To use different endpoint policies for different workloads, use separate VPCs.

## Guardrail 7: Security Groups on Endpoint ENIs Must Allow Inbound Traffic
Interface endpoint ENIs have security groups that control inbound traffic. The security group must allow inbound traffic on the service port (e.g., 443 for HTTPS) from the client CIDR or security group. The default VPC security group may not allow the required traffic. Both the client's outbound and the endpoint's inbound security group rules must permit the connection.

## Guardrail 8: Endpoint Service Connections Require Acceptance
By default, connections to an endpoint service require manual acceptance by the service provider. The connection stays in "pendingAcceptance" state until accepted. Auto-accept can be enabled on the endpoint service. Pending connections do NOT pass traffic. Check connection status with describe-vpc-endpoint-connections.

## Guardrail 9: Private DNS Changes Default Service DNS Resolution
When private DNS is enabled on an interface endpoint, the service's default public DNS name (e.g., ec2.us-east-1.amazonaws.com) resolves to the endpoint's private IPs instead of public IPs. This affects ALL resources in the VPC, not just specific instances. Disabling private DNS reverts to public IP resolution. This is the mechanism that enables transparent PrivateLink adoption.

## Guardrail 10: Endpoint ENIs Are Placed in Specified Subnets Only
When creating an interface endpoint, you specify which subnets (and thus AZs) to place ENIs in. Only one subnet per AZ can be specified. If you don't specify a subnet in an AZ, clients in that AZ must cross AZ boundaries to reach the endpoint. Adding or removing subnets after creation requires modifying the endpoint.

## Guardrail 11: Endpoint Service Permissions Control Who Can Create Endpoints
The endpoint service provider controls which AWS accounts or IAM principals can create endpoints to their service using service permissions (allowedPrincipals). If an account is not in the allowed list, endpoint creation will fail with an authorization error. AWS service endpoints (com.amazonaws.*) are available to all accounts by default.

## Guardrail 12: VPC Endpoint Lifecycle States Affect Connectivity
VPC endpoints go through lifecycle states: pending → available → deleting → deleted. Endpoints also have pendingAcceptance, rejected, expired, and failed states. Only endpoints in "available" state pass traffic. Modifying an endpoint (adding subnets, changing policy) may temporarily affect connectivity. Check the State field in describe-vpc-endpoints output.
