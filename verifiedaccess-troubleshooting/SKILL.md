---
name: verifiedaccess-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Verified Access problems
  by analyzing instance configuration, group policies, endpoint connectivity,
  trust providers, Cedar policy language, logging, networking, and following
  structured runbooks. Activate when: instance creation failures, trust provider
  configuration errors, group policy evaluation issues, endpoint creation
  failures, DNS resolution problems, endpoint connectivity issues, identity
  trust provider errors, device trust provider problems, Cedar policy syntax
  errors, policy debugging difficulties, access log analysis, CloudWatch
  integration issues, VPC configuration problems, security group errors, or
  the user says something is wrong with Verified Access without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with EC2 (Verified Access), IAM, CloudWatch,
  CloudTrail, and optionally Route 53 permissions. Trust provider configuration
  may require IdP admin access.
---

# AWS Verified Access Diagnostics

## When to use

Any AWS Verified Access investigation where the console alone is insufficient — instance setup, trust provider configuration, group policy evaluation, endpoint connectivity, Cedar policy debugging, access logging, or VPC networking issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-verified-access-instances --query 'VerifiedAccessInstances[*].{Id:VerifiedAccessInstanceId,Description:Description,TrustProviders:VerifiedAccessTrustProviders}'
aws ec2 describe-verified-access-groups --query 'VerifiedAccessGroups[*].{Id:VerifiedAccessGroupId,Instance:VerifiedAccessInstanceId,Description:Description}'
aws ec2 describe-verified-access-endpoints --query 'VerifiedAccessEndpoints[*].{Id:VerifiedAccessEndpointId,Group:VerifiedAccessGroupId,Type:EndpointType,Status:Status}'
aws ec2 describe-verified-access-trust-providers --query 'VerifiedAccessTrustProviders[*].{Id:VerifiedAccessTrustProviderId,Type:TrustProviderType,PolicyReference:PolicyReferenceName}'
```

### Step 2 — Domain deep dive

```
aws ec2 describe-verified-access-endpoints --verified-access-endpoint-ids <endpoint-id>
aws ec2 describe-verified-access-groups --verified-access-group-ids <group-id>
aws ec2 describe-verified-access-instance-logging-configurations --verified-access-instance-ids <instance-id>
aws ec2 describe-security-groups --group-ids <sg-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name <verified-access-log-group> --filter-pattern "DENY"
aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>
```

Read `references/verifiedaccess-guardrails.md` before concluding on any Verified Access issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-verified-access-instances` | Check instance configuration and trust providers |
| `describe-verified-access-groups` | Check group configuration and policies |
| `describe-verified-access-endpoints` | Check endpoint status, DNS, connectivity |
| `describe-verified-access-trust-providers` | Check trust provider configuration |
| `get-verified-access-group-policy` | Retrieve Cedar policy for a group |
| `describe-verified-access-instance-logging-configurations` | Check logging setup |
| CloudWatch Logs | Analyze access logs and policy decisions |
| CloudTrail | Audit configuration changes |

## Gotchas: AWS Verified Access

- Verified Access uses Cedar policy language, NOT IAM policy JSON. Cedar is a separate policy language with its own syntax. Cedar policies use `permit` and `forbid` statements with `when` clauses. Do not confuse Cedar with IAM JSON policies. Cedar policies are attached to Verified Access groups, not endpoints.
- Trust providers supply identity and device context. Identity trust providers (IAM Identity Center, OIDC) provide user identity claims. Device trust providers (CrowdStrike, Jamf) provide device posture. Both types can be attached to a single Verified Access instance. Policy decisions use context from trust providers.
- Endpoints are the access points for applications. Each endpoint maps to a specific application (load balancer or network interface). Endpoints belong to a group and inherit the group's policy. Endpoint DNS must be configured for users to reach the application through Verified Access.
- Verified Access operates at the application layer (HTTP/HTTPS). It is NOT a network-level firewall. It evaluates policies based on identity and device context for each HTTP request. It does not inspect or filter network packets. For network-level controls, use security groups and NACLs.
- Logging must be explicitly enabled. Access logs are not enabled by default. Logs can be sent to CloudWatch Logs, S3, or Kinesis Data Firehose. Logs contain policy evaluation results, user identity, device context, and request details. Enable logging for troubleshooting and compliance.
- DNS configuration is required for endpoints. Each endpoint gets a Verified Access DNS name. You must create a CNAME record pointing your application domain to the endpoint DNS name. Without proper DNS, users cannot reach the application through Verified Access.

## Anti-hallucination rules

1. Always cite specific instance IDs, group IDs, or API responses as evidence.
2. Cedar policy language is NOT IAM JSON. Never suggest IAM policy syntax for Verified Access.
3. Trust providers are NOT IAM roles. Never confuse identity providers with IAM.
4. Verified Access is application-layer, NOT network-layer. Never suggest it replaces firewalls.
5. Policies are on groups, NOT endpoints. Never suggest attaching policies to endpoints.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1–A2 | Instance creation, trust provider config |
| B — Group | B1–B2 | Group policies, policy evaluation |
| C — Endpoint | C1–C3 | Endpoint creation, DNS, connectivity |
| D — Trust | D1–D2 | Identity trust provider, device trust provider |
| E — Policy | E1–E2 | Cedar policy language, policy debugging |
| F — Logging | F1–F2 | Access logs, CloudWatch integration |
| G — Networking | G1–G2 | VPC config, security groups |
| Z — Catch-All | Z1 | General troubleshooting |
