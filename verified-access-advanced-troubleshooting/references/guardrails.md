# AWS Verified Access Advanced Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Verified Access issue.

## Guardrail 1: Cedar Is Not IAM Policy Language
Verified Access uses Cedar policy language for authorization decisions. Cedar has its own syntax with `permit`, `forbid`, `when`, and `unless` clauses. It is NOT IAM policy JSON. Do not provide IAM policy syntax when Cedar is required.

## Guardrail 2: Trust Provider Types Are Distinct
User-trust providers (OIDC, IAM Identity Center) and device-trust providers (CrowdStrike, Jamf) provide different context data. User-trust provides identity claims (email, groups). Device-trust provides device posture (compliance, OS version). Do not conflate the two.

## Guardrail 3: Policy Reference Names Must Match
Cedar policies reference trust provider data using the `policy_reference_name` configured on the trust provider. If the name in the policy does not match the trust provider's configured name, the context data will not be available and the policy will fail silently.

## Guardrail 4: WAF Evaluates Before Verified Access
When WAF is associated with a Verified Access instance, WAF rules are evaluated BEFORE Verified Access Cedar policies. A WAF block prevents the request from reaching Verified Access policy evaluation. Debug WAF first if requests are blocked.

## Guardrail 5: Access Logs Are Essential for Debugging
Verified Access access logs contain trust context, policy evaluation results, and denial reasons. Always enable and check access logs before debugging policy issues. Without logs, policy debugging is guesswork.

## Guardrail 6: Endpoint Types Have Different Requirements
Load-balancer endpoints require an ALB/NLB in the same VPC. Network-interface endpoints connect to specific ENIs. RDS endpoints connect to RDS instances. Each type has different security group, subnet, and routing requirements.

## Guardrail 7: Multiple Trust Providers Combine Context
When multiple trust providers are attached to an instance, Cedar policies can reference context from ALL providers. If any provider is unavailable or returns incomplete data, policies depending on that data will fail.

## Guardrail 8: DNS Configuration Is Critical
Verified Access endpoints require DNS configuration to route traffic through the Verified Access infrastructure. Incorrect DNS (missing CNAME, wrong endpoint domain) causes connectivity failures that look like application issues.

## Guardrail 9: Security Groups Apply to Endpoints
Verified Access endpoints have associated security groups that control network-level access. Even if Cedar policies permit access, security group rules can block traffic. Check both layers.

## Guardrail 10: OIDC Token Claims Vary by Provider
Different OIDC providers include different claims in their tokens. Cedar policies that reference claims not present in the token will fail. Verify the actual token claims from your OIDC provider before writing policies.

## Guardrail 11: Verified Access Is Regional
Verified Access instances, groups, and endpoints are regional resources. They do not span regions. Cross-region access requires separate Verified Access deployments in each region.

## Guardrail 12: Certificate Requirements for Endpoints
Verified Access endpoints serving HTTPS traffic require valid TLS certificates. Certificate issues (expired, wrong domain, untrusted CA) cause connection failures that may be misdiagnosed as Verified Access policy issues.
