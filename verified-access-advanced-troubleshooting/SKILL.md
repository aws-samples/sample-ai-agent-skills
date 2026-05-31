---
name: verified-access-advanced-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Verified Access
  advanced problems by analyzing instance creation, trust provider
  configuration, Cedar policy debugging, endpoint connectivity, device
  and user trust evaluation, access logs, WAF integration, and
  multi-trust-provider patterns using structured runbooks.
  Activate when: instance creation failures, trust provider issues,
  Cedar policy errors, endpoint connectivity problems, device trust
  failures, access log analysis, WAF integration issues, or the user
  says something is wrong with Verified Access.
compatibility: >
  Requires AWS CLI or SDK access with ec2 (Verified Access APIs),
  logs, wafv2, sso, iam, and networking permissions.
---

# AWS Verified Access Advanced Diagnostics

## When to use

Any AWS Verified Access advanced investigation — instance creation, trust provider configuration, Cedar policy debugging, endpoint group management, device and user trust evaluation, access log analysis, WAF integration, or multi-trust-provider patterns.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-verified-access-instances --query 'VerifiedAccessInstances[*].{Id:VerifiedAccessInstanceId,Status:CreationTime,TrustProviders:VerifiedAccessTrustProviders}'
aws ec2 describe-verified-access-trust-providers --query 'VerifiedAccessTrustProviders[*].{Id:VerifiedAccessTrustProviderId,Type:TrustProviderType,PolicyReferenceName:PolicyReferenceName}'
aws ec2 describe-verified-access-groups --query 'VerifiedAccessGroups[*].{Id:VerifiedAccessGroupId,InstanceId:VerifiedAccessInstanceId,PolicyDocument:PolicyDocument}'
```

### Step 2 — Domain deep dive

```
aws ec2 describe-verified-access-endpoints --query 'VerifiedAccessEndpoints[*].{Id:VerifiedAccessEndpointId,GroupId:VerifiedAccessGroupId,Type:EndpointType,Status:Status}'
aws ec2 get-verified-access-group-policy --verified-access-group-id <group-id>
aws ec2 describe-verified-access-instance-logging-configurations --verified-access-instance-ids <instance-id>
```

### Step 3 — Detailed investigation

```
aws logs filter-log-events --log-group-name <verified-access-log-group> --filter-pattern '{ $.activity = "denied" }' --start-time <epoch-ms> --end-time <epoch-ms>
aws wafv2 get-web-acl-for-resource --resource-arn <verified-access-instance-arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Verified Access issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ec2 describe-verified-access-instances` | List VA instances |
| `ec2 describe-verified-access-trust-providers` | List trust providers |
| `ec2 describe-verified-access-groups` | List endpoint groups |
| `ec2 describe-verified-access-endpoints` | List endpoints |
| `ec2 get-verified-access-group-policy` | Get Cedar policy |
| `ec2 describe-verified-access-instance-logging-configurations` | Check logging config |
| `logs filter-log-events` | Analyze access logs |

## Gotchas: AWS Verified Access

- Verified Access uses Cedar policy language, NOT IAM policy JSON. Cedar has different syntax, operators, and evaluation logic. Do not confuse the two.
- Trust providers can be user-trust (OIDC/IAM Identity Center) or device-trust (CrowdStrike, Jamf, etc.). Each type provides different context data to Cedar policies.
- Policy evaluation uses the trust data from ALL attached trust providers. If a trust provider is unavailable, the context data it provides will be missing, potentially causing denials.
- Endpoint types include load-balancer, network-interface, and RDS. Each type has different connectivity requirements and configuration patterns.
- Access logs contain detailed trust context and policy evaluation results. Always check logs before debugging policies.
- WAF can be associated with Verified Access instances for additional security layers. WAF evaluation happens BEFORE Verified Access policy evaluation.
- Cedar policies reference trust provider data using the policy_reference_name configured on the trust provider. Mismatched names cause silent policy failures.

## Anti-hallucination rules

1. Always cite specific Verified Access instance IDs, group IDs, or access log entries as evidence.
2. Cedar policy language is NOT IAM policy JSON. Never provide IAM policy syntax when Cedar is needed.
3. Trust provider types (user-trust vs device-trust) provide different context. Never conflate them.
4. WAF evaluation happens BEFORE Verified Access policy evaluation. Never reverse the order.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance & Trust | A1–A2 | Instance creation failures, trust provider advanced |
| B — Policies | B1–B2 | Cedar policy debugging, policy evaluation errors |
| C — Endpoints | C1–C2 | Endpoint group configuration, endpoint connectivity |
| D — Trust Evaluation | D1–D2 | Device trust integration, user trust evaluation |
| E — Observability & WAF | E1–E2 | Access log analysis, WAF integration |
| F — Advanced | F1 | Multi-trust-provider |
| Z — Catch-All | Z1 | General troubleshooting |
