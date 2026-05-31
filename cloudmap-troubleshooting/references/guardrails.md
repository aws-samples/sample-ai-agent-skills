# AWS Cloud Map Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Cloud Map issue.

## Guardrail 1: DNS Namespaces and HTTP Namespaces Are Fundamentally Different
DNS namespaces (public or private) create Route 53 hosted zones and DNS records. HTTP namespaces are API-only — no DNS records are created. Discovery for HTTP namespaces uses the DiscoverInstances API exclusively. Never suggest DNS resolution for HTTP namespace services.

## Guardrail 2: DNS Records Are Managed Automatically — Do Not Modify Manually
For DNS namespaces, Cloud Map automatically creates and removes Route 53 records when instances register and deregister. Manual modification of these records can cause conflicts and inconsistencies. Always use Cloud Map APIs to manage instances, not Route 53 directly.

## Guardrail 3: HTTP Namespaces Use API-Only Discovery
HTTP namespaces do not create DNS records. Service discovery is performed exclusively through the DiscoverInstances API. This is useful for non-IP-based discovery or when DNS is not needed. Do not expect DNS resolution to work for HTTP namespace services.

## Guardrail 4: Route 53 Health Checks Only Work for Public Endpoints
Route 53 health checks can only reach publicly accessible endpoints. For private endpoints (in VPCs), use custom health checks where your application reports health status via UpdateInstanceCustomHealthStatus. Never configure Route 53 health checks for private IP addresses.

## Guardrail 5: Custom Health Checks Require Application-Managed Status
Cloud Map custom health checks do NOT automatically probe endpoints. Your application must call UpdateInstanceCustomHealthStatus to report HEALTHY or UNHEALTHY status. Without these calls, the health status remains at its initial value. Cloud Map does not check health for you.

## Guardrail 6: ECS-Managed Cloud Map Resources Should Not Be Modified Directly
When ECS service discovery is enabled, ECS creates and manages Cloud Map services and instances automatically. Manual modification (registering/deregistering instances, changing service config) can conflict with ECS management and cause inconsistencies. Let ECS manage its Cloud Map resources.

## Guardrail 7: DNS TTL Directly Affects Discovery Latency
DNS TTL determines how long clients cache DNS responses. Lower TTL means faster discovery of changes but more DNS queries. During deployments, high TTL causes clients to use stale IP addresses. Default TTL is 60 seconds. Adjust based on deployment frequency and tolerance for stale data.

## Guardrail 8: Instance Attributes Are for API Discovery Only
Instance attributes (key-value pairs) are used for filtering in DiscoverInstances API calls. They are NOT reflected in DNS records. DNS discovery only returns IP addresses (A records) or port/IP combinations (SRV records). Use API discovery if attribute-based filtering is needed.

## Guardrail 9: Namespace Deletion Requires Cascading Manual Cleanup
Namespaces cannot be deleted while they contain services. Services cannot be deleted while they contain instances. You must deregister all instances, delete all services, then delete the namespace. There is no cascading delete operation.

## Guardrail 10: Private DNS Namespaces Require VPC Association
Private DNS namespaces create Route 53 private hosted zones. These zones must be associated with the VPCs where clients need to resolve DNS. Without VPC association, DNS queries for the namespace will fail. Multiple VPCs can be associated with the same private hosted zone.

## Guardrail 11: Cloud Map Operations Are Eventually Consistent
Instance registration, deregistration, and health status changes are eventually consistent. Changes may take a few seconds to propagate through the system. DNS changes additionally depend on TTL expiration at the client. Do not expect instant discovery updates.

## Guardrail 12: Service Quotas Limit Scale
Cloud Map has quotas for instances per service (default 1,000), services per namespace (default 100), and namespaces per account (default 50). Check and request quota increases before large-scale deployments. Exceeding quotas causes registration failures.
