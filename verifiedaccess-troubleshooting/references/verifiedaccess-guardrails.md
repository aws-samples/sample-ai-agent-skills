# AWS Verified Access Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Verified Access issue.

## Guardrail 1: Cedar Policy Language Is NOT IAM JSON
Verified Access uses Cedar policy language, a separate language from IAM JSON policies. Cedar uses `permit` and `forbid` keywords with `when` clauses. Cedar policies reference trust provider context (e.g., `context.identity.email`). Do NOT suggest IAM policy JSON syntax for Verified Access policies. Cedar has its own operators, types, and evaluation logic.

## Guardrail 2: Trust Providers Supply Context, Not Permissions
Trust providers (identity and device) supply context data for policy evaluation. Identity providers (OIDC, IAM Identity Center) provide user claims (email, groups). Device providers (CrowdStrike, Jamf) provide device posture (compliance, OS version). Trust providers do NOT grant or deny access directly — policies use their context to make decisions.

## Guardrail 3: Policies Are Attached to Groups, NOT Endpoints
Cedar policies are configured on Verified Access groups. Endpoints belong to groups and inherit the group's policy. You cannot attach a policy directly to an endpoint. To apply different policies to different applications, place them in different groups. All endpoints in a group share the same policy.

## Guardrail 4: Verified Access Is Application-Layer, NOT Network-Layer
Verified Access evaluates policies at the HTTP/HTTPS application layer. It is NOT a network firewall, security group, or NACL replacement. It does not inspect network packets or filter by IP/port. For network-level controls, continue using security groups, NACLs, and AWS Network Firewall. Verified Access complements, not replaces, network security.

## Guardrail 5: Access Logging Must Be Explicitly Enabled
Verified Access does not log access decisions by default. Logging must be enabled on the Verified Access instance. Logs can go to CloudWatch Logs, S3, or Kinesis Data Firehose. Without logging, you cannot see why access was denied or allowed. Always enable logging before troubleshooting policy issues.

## Guardrail 6: DNS CNAME Is Required for Endpoint Access
Each Verified Access endpoint has a generated DNS name. Users access applications through your custom domain, which must CNAME to the endpoint DNS name. Without the CNAME, users cannot reach the application through Verified Access. The endpoint DNS name is in the format: <endpoint-id>.prod.verified-access.<region>.amazonaws.com.

## Guardrail 7: Policy Evaluation Uses Explicit Deny by Default
If no policy permits the request, access is denied (implicit deny). A `forbid` statement overrides any `permit` statement (explicit deny wins). Policies are evaluated for the group the endpoint belongs to. An empty policy (no statements) denies all access. Always have at least one `permit` statement for legitimate access.

## Guardrail 8: Groups Belong to Exactly One Instance
Each Verified Access group belongs to exactly one Verified Access instance. Groups cannot be shared across instances. Endpoints belong to exactly one group. The hierarchy is: Instance → Groups → Endpoints. Trust providers are attached at the instance level and available to all groups in that instance.

## Guardrail 9: Endpoint Types Determine Backend Connectivity
Load balancer endpoints connect to ALB/NLB in your VPC. Network interface endpoints connect to a specific ENI. The endpoint type determines how Verified Access reaches your application. Load balancer endpoints require the LB to be in the same VPC. Network interface endpoints connect to a specific IP.

## Guardrail 10: Security Groups Control Network Access to Endpoints
Verified Access endpoints have security groups that control network-level access. The security group must allow inbound HTTPS (443) traffic. The endpoint's security group must also allow outbound traffic to the backend application. Security groups are separate from Cedar policies — both must allow access.

## Guardrail 11: A Verified Access Instance Can Have Multiple Trust Providers
A single Verified Access instance can have one identity trust provider AND one device trust provider attached simultaneously. This enables policies that evaluate both user identity AND device posture. You cannot attach multiple identity providers or multiple device providers to the same instance.

## Guardrail 12: Cedar Policy Context References Use Dot Notation
Cedar policies reference trust provider context using dot notation: `context.<policy-reference-name>.<claim>`. The policy reference name is set when creating the trust provider. For OIDC: `context.myoidc.email`, `context.myoidc.groups`. For device: `context.mydevice.is_compliant`. Incorrect context paths cause policy evaluation failures (treated as missing data, resulting in deny).
