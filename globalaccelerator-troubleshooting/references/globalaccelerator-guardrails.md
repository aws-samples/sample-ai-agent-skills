# AWS Global Accelerator Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Global Accelerator issue.

## Guardrail 1: Static Anycast IPs — Exactly 2 Per Accelerator, Immutable
Each accelerator receives exactly 2 static anycast IP addresses from the AWS global network. These IPs cannot be changed, released, or reassigned after creation. They persist for the lifetime of the accelerator. If you need different IPs, you must create a new accelerator. BYOIP (Bring Your Own IP) is supported as an alternative.

## Guardrail 2: Endpoint Health Checks Are Independent of ELB Health Checks
Global Accelerator performs its own health checks on endpoints. These are separate from any ELB health checks. An ALB target can be healthy in the ALB target group but unhealthy in Global Accelerator if the GA health check port, path, or protocol differs. Always check both GA and ELB health check configurations independently.

## Guardrail 3: Traffic Dial Is Per-Endpoint-Group, Not Per-Endpoint
The traffic dial controls the percentage of traffic routed to an entire endpoint group (which maps to a region). It does NOT control traffic to individual endpoints within a group. To control per-endpoint traffic distribution, use endpoint weights (0-255). A traffic dial of 0% stops all traffic to that region's endpoint group.

## Guardrail 4: Client Affinity Is Per-Listener, Not Per-Endpoint
Client affinity (SOURCE_IP) is configured at the listener level. When enabled, all requests from the same client IP are routed to the same endpoint for the duration of the connection. It applies to ALL endpoint groups under that listener. You cannot set different affinity per endpoint group. Default is NONE.

## Guardrail 5: Custom Routing Accelerators Are a Separate Resource Type
Custom routing accelerators are different from standard accelerators. They map specific ports to specific EC2 instances in VPC subnets. Custom routing does NOT support ALB, NLB, or EIP endpoints — only EC2 instances via subnet. Use different API calls (list-custom-routing-accelerators, etc.). Do not mix standard and custom routing APIs.

## Guardrail 6: Global Accelerator Is NOT a CDN — No Caching
Global Accelerator optimizes the network path using the AWS global network backbone. It does NOT cache content like CloudFront. Every request is forwarded to the endpoint. For content caching, use CloudFront. GA and CloudFront can be used together — GA for TCP/UDP optimization, CloudFront for HTTP caching.

## Guardrail 7: DDoS Protection — Shield Standard Included, Shield Advanced Optional
AWS Shield Standard protection is automatically included with Global Accelerator at no extra cost. For enhanced DDoS protection, incident response, and cost protection, associate Shield Advanced. Shield Advanced requires a separate subscription ($3,000/month). Do not claim Shield Advanced is free or automatic.

## Guardrail 8: Endpoint Types and Limits
Supported endpoint types: ALB, NLB, EC2 instances, and Elastic IP addresses. Each endpoint group supports up to 10 endpoints. Endpoints must be in the same AWS region as the endpoint group. EC2 instances must have public IP addresses or be behind an ALB/NLB. Internal-only resources cannot be direct endpoints.

## Guardrail 9: Cross-Account Endpoints Require Attachment Resources
Cross-account endpoints require creating a cross-account attachment resource. The endpoint account must accept the attachment. This is not automatic — both accounts must take action. Without proper cross-account attachment, adding endpoints from another account will fail.

## Guardrail 10: Flow Logs Are Published to S3, Not CloudWatch Logs
Global Accelerator flow logs are published to an S3 bucket, not CloudWatch Logs. Flow logs capture source/destination IPs, ports, protocol, packets, bytes, and action. They must be explicitly enabled via accelerator attributes. There is an S3 storage cost for flow logs.

## Guardrail 11: Listener Port Ranges and Protocol Restrictions
Each listener supports port ranges (e.g., 80-80, 443-443, or 1024-65535). Protocols supported are TCP and UDP. A single listener can have multiple port ranges. Port ranges across listeners on the same accelerator cannot overlap. Maximum of 10 listeners per accelerator.

## Guardrail 12: Accelerator Changes Take Time to Propagate
Changes to accelerator configuration (enabling/disabling, endpoint changes, listener updates) take time to propagate across the AWS global network. The accelerator status shows IN_PROGRESS during propagation. Do not assume changes are effective immediately. Typical propagation is a few minutes but can take longer.
