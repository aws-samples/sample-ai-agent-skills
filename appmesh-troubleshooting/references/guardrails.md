# AWS App Mesh Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS App Mesh issue.

## Guardrail 1: App Mesh Is the Control Plane — Envoy Is the Data Plane
App Mesh manages configuration and pushes it to Envoy sidecar proxies. App Mesh does NOT route traffic itself. If Envoy sidecars are not running or cannot connect to the App Mesh control plane (Envoy Management Service), traffic routing will not work regardless of mesh configuration. Always verify Envoy sidecar presence and health first.

## Guardrail 2: Virtual Nodes Require Service Discovery
Virtual nodes represent actual compute resources and must have a service discovery mechanism configured — either DNS (Route 53) or AWS Cloud Map. Without service discovery, other virtual nodes cannot find and route traffic to the target. Verify service discovery configuration before investigating routing issues.

## Guardrail 3: Virtual Service Providers Must Be Explicitly Configured
A virtual service does not automatically route to a virtual node. You must configure a provider — either a virtual router (for routing rules) or a virtual node (for direct routing). A virtual service without a provider will not route traffic. Always check the provider configuration.

## Guardrail 4: Envoy Sidecar Must Be Injected Into Every Mesh Service
Traffic only flows through the mesh if Envoy sidecars are present. On EKS, the App Mesh controller handles injection via annotations. On ECS, the Envoy container must be in the task definition. Missing sidecars mean traffic bypasses the mesh entirely — the application works but without mesh features.

## Guardrail 5: mTLS Requires ACM Private CA — Not Public ACM Certificates
App Mesh mTLS uses certificates from AWS Certificate Manager Private Certificate Authority (ACM-PCA). Public ACM certificates cannot be used for mTLS. The private CA must be active and the mesh must be configured with the correct CA ARN. Certificate rotation is automatic but CA configuration must be correct.

## Guardrail 6: Route Priority Matters — More Specific Routes First
Routes in a virtual router are evaluated in order. More specific routes (exact path match) should have higher priority than less specific routes (prefix match). Incorrect ordering causes traffic to match the wrong route. Always review route priority when traffic goes to unexpected destinations.

## Guardrail 7: App Mesh Health Checks Are Separate from Container Health Checks
App Mesh virtual node health checks determine if a node is healthy for mesh routing. ECS/EKS container health checks determine if the container is running. These are independent — a container can be healthy but the virtual node unhealthy (or vice versa). Check both when investigating connectivity issues.

## Guardrail 8: Retry Policies and Timeouts Are Per-Route Configuration
Retries and timeouts are configured on individual routes, not globally. Default timeouts (15 seconds) may be too short for slow backends. Retry policies should specify appropriate retry conditions (5xx, gateway-error, etc.). Missing retry configuration means no automatic retries on failure.

## Guardrail 9: Cross-Account Mesh Sharing Requires AWS RAM
Meshes can be shared across AWS accounts using AWS Resource Access Manager (RAM). The mesh owner shares the mesh, and consumer accounts create virtual nodes/services in the shared mesh. Without RAM sharing, cross-account mesh operations will fail with access denied.

## Guardrail 10: Envoy Admin API Is the Source of Truth for Proxy State
The Envoy admin API (port 9901) provides real-time proxy configuration and state. The /config_dump endpoint shows the actual Envoy configuration. The /clusters endpoint shows upstream cluster health. The /stats endpoint shows traffic metrics. Always check the admin API when App Mesh configuration doesn't match observed behavior.

## Guardrail 11: App Mesh Controller for EKS Manages CRDs
The App Mesh controller translates Kubernetes custom resources (Mesh, VirtualNode, VirtualService, etc.) into App Mesh API calls. Controller issues (crashes, permission errors, version mismatches) affect all mesh operations on EKS. Always check controller pod health and logs when EKS mesh operations fail.

## Guardrail 12: Traffic Shifting Weights Must Be Valid
Weighted routes for traffic shifting (canary, blue/green) require valid weight configurations. All target virtual nodes must be healthy. Weights of 0 are valid (no traffic). If a target virtual node is unhealthy, traffic may not shift as expected. Always verify target node health during traffic shifting.
